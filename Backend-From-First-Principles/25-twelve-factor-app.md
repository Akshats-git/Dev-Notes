# 25 — The Twelve-Factor App

**Video:** [#27 The Twelve-Factor App](https://www.youtube.com/watch?v=2_fcv3-XfN4) (1h 33m)

Throughout the series the speaker prescribed things without defending them: config in environment variables, the server binds a port, logs go to stdout, processes are stateless, killing a process loses nothing. This video explains where those decisions come from: the **12-Factor App**.

> A twelve-factor app is a **contract between your application and the platform that runs it**. It states what must be true about your app so any machine, platform or new teammate can run, scale and update it **without surprises**.

Source: **12factor.net**, written in **2011** by Heroku's team (Adam Wiggins, co-founder), one of the first PaaS companies, who had watched thousands of apps deploy, scale and break. The document was frozen for years (last small update 2017). In **Nov 2024** Heroku (Salesforce) open-sourced it under a neutral org with maintainers from AWS and Google Cloud. A rewrite is in progress on a `next` branch. The speaker covers the 2011 text first, then what the rewrite changes.

**Scope:** apps that run as network **services** (web apps, APIs) in any language. Not desktop/mobile apps, and not general principles like DRY/KISS.

---

## Contents

1. [Why it existed: snowflake servers and software erosion](#why-it-existed-snowflake-servers-and-software-erosion)
2. [Factor 1: Codebase](#factor-1-codebase)
3. [Factor 2: Dependencies](#factor-2-dependencies)
4. [Factor 3: Config](#factor-3-config)
5. [Factor 4: Backing services](#factor-4-backing-services)
6. [Factor 5: Build, release, run](#factor-5-build-release-run)
7. [Factor 6: Processes](#factor-6-processes)
8. [Factor 7: Port binding](#factor-7-port-binding)
9. [Factor 8: Concurrency](#factor-8-concurrency)
10. [Factor 9: Disposability](#factor-9-disposability)
11. [Factor 10: Dev/prod parity](#factor-10-devprod-parity)
12. [Factor 11: Logs](#factor-11-logs)
13. [Factor 12: Admin processes](#factor-12-admin-processes)
14. [The twelve at a glance](#the-twelve-at-a-glance)
15. [Summary](#summary)
16. [Interview questions](#interview-questions)

---

## Why it existed: snowflake servers and software erosion

[7:18](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=438s)

Deploying circa 2011:

1. Rent a server (or keep a physical machine in the office).
2. Install the OS **by hand**, then the language runtime and DB, then copy the app (often via **FTP**).
3. Edit config files on the server over **SSH**.

Every server became a **snowflake**: unique, hand-built, **impossible to reproduce**. If it died, you rebuilt from memory or a wiki. New developers took days to get a local copy, and it never matched production ("works on my machine"). Deploy days froze all other work.

The 12-Factor term for the resulting rot is **software erosion**: the app eventually stops working although nobody changed the code. For example, the OS is patched, a second app on the same server updates a shared library, or the one person who set it up leaves without documenting. The twelve factors are the defences.

---

## Factor 1: Codebase

*One codebase tracked in revision control, many deploys.* [15:04](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=904s)

- **Tracked in revision control.** Obvious now (Git), but in 2011 production code was often a folder or `final_v2_really_final.zip`.
- **Many deploys.** A **deploy** is one running instance (production, staging, a PR preview, your laptop). **One codebase, many deploys.** What differs between deploys is **never the code** but the **config**.

Example: the repo is at commit 100. Production runs 97, staging runs 99, your laptop is at 100 plus 2 unpushed commits. Four deploys, one codebase. On release day, staging then production move to 100.

Boundaries:

- Multiple codebases = a **distributed system**, with each app following the factors on its own.
- Shared code by **copy-paste** is a violation. Extract a library and consume it via a dependency manager (Factor 2).
- **Monorepos** are *not* an anti-pattern: a monorepo building 5 apps is 5 codebases that happen to live in one repo. The anti-pattern is the **same app existing in two places**.

---

## Factor 2: Dependencies

*Explicitly declare and isolate dependencies.* [21:18](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=1278s)

| Part | Meaning | Examples |
|---|---|---|
| **Declare** | A manifest listing every dependency | `package.json`, `go.mod`, `requirements.txt` / `pyproject.toml`, `Gemfile` |
| **Pin** (modern addition) | **Lock files** pin exact versions (including transitive deps) | `package-lock.json`, `go.sum` |
| **Isolate** | Nothing leaks in from the host machine | Python `virtualenv`, Ruby `bundle exec`; **containers** |

Why both: if you declare but don't isolate, Python may import the **system's** `requests` v1 instead of your declared v2. If you isolate but don't declare, you can't reproduce the set.

Subtle case: shelling out to **ImageMagick**. On a Mac with Homebrew it's v7 (binary `magic`); on Ubuntu it's v6 (binary `convert`, different defaults). Same code, different behaviour, because system tools are undeclared dependencies.

**Modern view:** **containers** freeze libraries *and* the OS, system tools and runtime, the peak of declaration + isolation. "Docker practically solved this factor."

---

## Factor 3: Config

*Store config in the environment.* [28:27](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=1707s)

**Config = everything likely to differ between deploys:** DB URL (local Docker container vs RDS), email provider (sandbox vs real), object storage (local MinIO vs S3), Redis, credentials.

**The test** from the document: *"Could you open-source your codebase right now without compromising a single credential?"* If not, config is hiding in the code.

**Why environment variables:**

1. Easy to change per deploy without touching files.
2. Can't be accidentally checked into the repo (how credentials get leaked daily).
3. An **OS-level standard**, language/framework-agnostic (Go, Node, Rust, Python, bash). Alternatives were framework-specific (Java `.properties`, Rails `config/database.yml`).

Example from the author's `tasker` app: one config struct (server, database, auth, Redis, email, AWS, observability, cron) with **38 values**, all loaded from `TASKER_*` env vars. Git history shows config changed constantly while the code that reads it barely changed, exactly the point: things with different rates of change shouldn't live together.

**Don't group config into named environments** (`config/staging.rb`, `production.rb`...). It doesn't scale: need a "staging-like env with a different DB" and you add yet another file. Keep each setting an **independent, granular** variable and **compose** a deployment by setting values, not by choosing a mode.

### The secrets controversy

[34:38](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=2078s)

Env vars suit a DB *host*. But for *passwords*? A 2017 article by **Diogo Mónica** (ex-Docker security) argued **not to use env vars for secrets** because the environment **leaks**: child processes inherit it, crash reporters and debug pages dump it, it's readable at `/proc/<pid>/environ`, and tools log it.

**Verdict:** separating config from code is non-negotiable. Env vars are the right default for non-secrets. For credentials, as you grow, use a **secret manager** (Vault, AWS Parameter Store): fetch at startup, keep in memory, rotate, audit. Env vars injected by your platform's secrets feature are fine for a solo dev or small team (better than committed credentials).

---

## Factor 4: Backing services

*Treat backing services as attached resources.* [38:16](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=2296s)

A **backing service** is anything the app consumes over the network: Postgres, Redis, a queue, SMTP, S3, a payment API.

> Your code must not distinguish between a service **you run** and one **someone else runs**. Each is an **attached resource**, reached via a **resource handle** (URL + credentials) that lives in config.

A Postgres handle: `postgres://user:password@host:5432/dbname` (scheme, credentials, host, port, DB name).

**Test:** you should be able to swap local Postgres for **Neon, PlanetScale, Supabase or Amazon RDS** by changing only the handle: zero code changes.

In 2011 this was eccentric (DB on the same box as the app was the norm). Now it's universal, and DB + app on one machine is an anti-pattern. This factor "aged very well" and was nearly a prediction.

---

## Factor 5: Build, release, run

*Strictly separate build and run stages.* [42:59](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=2579s)

| Stage | What happens | Output |
|---|---|---|
| **Build** | Pull the code **at a commit**, fetch dependencies from the manifest, compile. **No config** | A **build** (a single binary for Go; app + `node_modules` for Node; a **Docker image**) |
| **Release** | **Build + the deploy's config** | A **release** (prod build + prod config = prod release; same build + staging config = staging release) |
| **Run** | Launch processes from a release | Running app |

**Why be strict:**

1. **Instant rollback.** Builds are immutable. Releases are an **append-only ledger**, each with an ID or timestamp and **never modified**.
   - Example: release 100 (build @97). Someone rotates the DB password → release **101** (same build, new config). You deploy new code → release **102**. Errors spike → **re-run release 101**: live in seconds. Debug the bad one at leisure.
2. **No hot-patching production.** Code only enters at the build stage; there's no path from the running machine back to the codebase. This ends the SSH-and-edit practice that made production drift away from the repo.

Rewrite addition: releases must be **immutable snapshots** (implied before, explicit now). Maps to Heroku releases, Docker image tags, Kubernetes rollouts.

---

## Factor 6: Processes

*Execute the app as one or more stateless processes.* [50:15](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=3015s)

- **Share nothing.** Anything that must persist goes into a stateful backing service.
- Assume **any request can be served by any process, and any process can disappear at any moment.**
- Memory and local disk are fine as a **cache for a single transaction** (e.g. a temp file while processing an upload) since nothing later depends on it.
- **Sticky sessions are a violation.** Pinning a user to an instance defeats the purpose of having several: kill an instance and everyone on it is logged out; add one and it gets no existing users. Put sessions in Redis/Postgres instead.

**Tension:** WebSockets hold state in process memory (see [23](23-real-time-backends.md)), solved with pub/sub. The speaker's take: statelessness is the default, but you may break it when **the feature itself is state** (real-time).

**Thumb rule:** if you can't kill any process at any moment without a user losing data, state is living where it doesn't belong.

---

## Factor 7: Port binding

*Export services via port binding.* [57:28](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=3448s)

In 2011 apps ran *inside* a web server: PHP as an Apache module; a Java app as a file dropped into Tomcat. The server's version and config became part of your runtime, so erosion again.

**The factor:** the app is **self-contained**. It bundles its HTTP server as a library (declared per Factor 2) and **binds to a port** (Go 8080, Node 3000/4000, DBs on their defaults). In dev: `localhost:8080`. In prod, a reverse proxy routes a public hostname to that port.

Consequence: **any app can be a backing service for another app** just by exposing a port.

**Aged:** **Serverless** (e.g. AWS Lambda) doesn't bind ports; the platform calls an exported function. The rewrite keeps port binding as the headline but generalises to "export your service through a declared interface".

---

## Factor 8: Concurrency

*Scale out via the process model.* [1:02:10](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=3730s)

Treat the **process as the unit of scaling**. In 2011 each runtime scaled differently (the JVM: one huge process with threads; PHP/Apache: child processes on demand). The factor takes the **Unix daemon model**: different work gets different **process types**:

| Process type | Handles |
|---|---|
| **web** | HTTP requests |
| **worker** | background jobs (video encoding, resizing) |
| **scheduler/clock** | cron-like tasks (daily email at midnight) |

Each scales **independently** by running more copies.

**Example (food-delivery app, each process ≈ 500 MB):** at lunch you need 10 web + 2 workers = 6 GB. If web and workers were one big process, scaling to 10 would also run 10 workers (≈10 GB, 8 idle). At night: 5 web + 20 workers.

The set of types and counts (`web=3, worker=2`) is the **process formation**. It's what you write today as `replicas: 3` in a Kubernetes YAML, or in a Procfile.

Notes:

- Threads/goroutines/event loops **inside** a process are fine, but the **unit of scale is the process**, because one machine runs out of room and share-nothing processes spread across machines.
- A 12-factor app must **not daemonize** or write PID files. Process lifecycle is the **platform's** job (OS/systemd/Kubernetes).

**Aged:** orchestrators adopted it so thoroughly that we barely think about it, but it still applies when you run under `systemd`.

---

## Factor 9: Disposability

*Maximise robustness with fast startup and graceful shutdown.* [1:09:57](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=4197s)

Processes must be **startable and stoppable at a moment's notice** with nothing destructive happening. (See [17](17-graceful-shutdown.md).)

Why: the platform constantly kills and starts processes: deploys, scaling, rescheduling. 5 deploys/week × 10 instances = **50 stop/starts a week**. Process death is routine, not an incident.

- **Fast startup**: if boot takes 60 s, a rolling deploy of 10 instances takes 10 min and autoscaling is a minute late for spikes. Aim for a few seconds.
- **Graceful shutdown**: on `SIGTERM`, a **web** process stops accepting new requests, finishes in-flight ones, exits. A **worker** returns the current job to the queue.
- **Re-entrancy/idempotency:** a returned job must leave no dirty state. Wrap in a transaction or make it idempotent so running twice is safe.
- **Robust against sudden death** (OOM kill, hardware failure: no warning). **Crash-only design**: if you survive crashes, the crash path *is* the shutdown path.

---

## Factor 10: Dev/prod parity

*Keep development, staging and production as similar as possible.* [1:17:12](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=4632s)

Three gaps:

| Gap | Traditional | 12-factor |
|---|---|---|
| **Time** | Deploy weeks later, many changes at once | Deploy hours later |
| **Personnel** | Devs write, ops deploy | The people who write the code deploy it and see its errors |
| **Tools** | Dev: SQLite/Mac stack; Prod: Postgres/Linux | Same backing services everywhere |

**The tools gap is still alive.** ORMs let you develop on SQLite and run Postgres in prod, but behaviour differs. Example: Postgres `LIKE` is case-sensitive (`ILIKE` isn't), while SQLite's `LIKE` is case-insensitive for ASCII, so a search that works locally breaks in production **with no error or warning**.

**Advice:** same DB/cache/queue in every environment. With Docker it takes one command (it was a real excuse in 2011, not in 2026).

**Nuance for 2026:** managed services can't always be run locally. DynamoDB is proprietary, and emulators like LocalStack approximate but don't replicate. Managed services bring real benefits, so weigh the trade-off: if the benefits are large, accept the parity gap.

---

## Factor 11: Logs

*Treat logs as event streams.* [1:24:23](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=5063s)

A log is **not a file**. A file is just one output format. A log is a **time-ordered stream of events** with no beginning or end.

**The app:** writes every event, **unbuffered, to stdout**. Doesn't manage log files, paths, rotation or routing.
**The platform:** captures the stream and decides where it goes (file, a log drain like Loki/Datadog, Syslog in 2011).

Why:

1. No app code changes between environments. Routing is the platform's job.
2. Streams **compose**: on your laptop you see them live in the terminal. In a container the runtime tags each line with its source. In production the platform **merges all instances' streams into one time-ordered stream**, so you don't care which instance wrote what.

Modern: **observability** (logs + metrics + traces) goes beyond the 2011 text, which only covered logs. Treat metrics/traces as additions (see [16](16-logging-monitoring-observability.md)).

---

## Factor 12: Admin processes

*Run admin/management tasks as one-off processes.* [1:29:37](https://www.youtube.com/watch?v=2_fcv3-XfN4&t=5377s)

One-off tasks (DB migrations, a data-fix script, a console) should run in an **identical environment** to the app's regular processes: **same codebase at the same commit, same config, same dependency isolation.**

Why: in the old days people SSH'd into prod and ran a manual `UPDATE`/`DELETE` with no review and no record, causing drift between production and the documented/coded state.

Canonical example: **database migrations** ship with the code and run as a one-off before the app starts.

**Demo:** the `tasker` app on a Linux box, driven by a **Procfile** (Heroku's invention for declaring process types), started by a process manager (**Overmind**). Boot log: schema check → job server → server binds to its port, all in the order the factors imply. A second pane runs an **auto-archive** admin task from the same code/config/environment, then exits. `Ctrl+C` shows graceful shutdown: connection pool closes, the job server waits for workers, clean exit.

---

## The twelve at a glance

| # | Factor | One-liner |
|---|---|---|
| I | **Codebase** | One repo per app, many deploys |
| II | **Dependencies** | Declare (manifest + lock) and isolate (venv/container) |
| III | **Config** | In the environment, never in code. Secrets in a secret manager |
| IV | **Backing services** | Attached resources via URL handles. Swap without code changes |
| V | **Build, release, run** | Immutable builds; release = build + config; instant rollbacks |
| VI | **Processes** | Stateless, share-nothing. No sticky sessions |
| VII | **Port binding** | Self-contained; export HTTP by binding a port |
| VIII | **Concurrency** | Scale out by process type and count |
| IX | **Disposability** | Fast start, graceful stop, crash-safe |
| X | **Dev/prod parity** | Same services, small gaps, frequent deploys |
| XI | **Logs** | Unbuffered event stream to stdout |
| XII | **Admin processes** | One-off tasks in the same env and codebase |

---

## Summary

1. The 12-Factor App (Heroku, 2011; open-sourced 2024) is the contract between your **app** and the **platform**. Many "rules" in this series come from it.
2. It was a defence against **snowflake servers** and **software erosion**.
3. **Code** (1) in one repo, **dependencies** (2) declared + isolated, **config** (3) in the environment, **backing services** (4) as swappable URL handles.
4. **Pipeline** (5): immutable build + config = release; run it. Rollback = re-run an old release.
5. **Runtime** (6–9): stateless share-nothing processes, bind a port, scale by process type, be disposable (fast start, graceful stop, crash-safe).
6. **Operations** (10–12): dev/prod parity, logs as stdout streams, admin tasks as one-offs.
7. What aged: containers solve (2); secrets in env vars debated (3); serverless bends (7); orchestrators absorb (8); managed services challenge (10); observability extends (11).

## Interview questions

**Q: What is the 12-Factor App?**
A set of twelve principles from Heroku for building apps that are portable, scalable and easy to operate on cloud platforms. A contract between app and platform.

**Q: Why store config in the environment?**
Easy to vary per deploy, can't be committed by accident, and it's language-agnostic. For real secrets, prefer a secrets manager.

**Q: What's the difference between a build, a release and a run?**
Build = code + dependencies at a commit. Release = a build + a deploy's config. Run = executing processes from a release.

**Q: How do 12-factor apps enable instant rollback?**
Releases are immutable and versioned. Redeploying a previous release (same build + same config) is just launching it again.

**Q: Why are sticky sessions discouraged?**
They tie users to an instance, so losing or adding instances breaks sessions and defeats horizontal scaling. Store session state in a backing service.

**Q: What does "treat backing services as attached resources" mean?**
The app addresses all services via a URL/handle from config, so swapping local Postgres for RDS needs no code change.

**Q: What is disposability?**
Processes start fast, shut down gracefully on SIGTERM and tolerate abrupt kills, so the platform can scale and deploy freely.

**Q: What is a process formation?**
The mapping of process types to counts (`web=3, worker=2`), e.g. Kubernetes replicas.

**Q: Why log to stdout?**
The app shouldn't manage log routing. The platform captures the stream and merges it across instances.

**Q: Why run migrations as one-off processes from the same codebase?**
Same code, config and dependencies ensure the task behaves like the app, and it's versioned and reviewed rather than ad-hoc SSH edits.

**Q: Does Factor 1 forbid monorepos?**
No. A monorepo with several apps is several codebases in one repo. The violation is the *same* app existing in multiple codebases (copy-paste).

**Q: Where is the 12-factor model strained today?**
Serverless (no port binding/long-running process), real-time (stateful connections), managed cloud services (dev/prod parity), secrets in env vars, and observability beyond logs.
