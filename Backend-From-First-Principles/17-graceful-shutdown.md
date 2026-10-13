# 17 — Graceful Shutdown

**Video:** [#19 Graceful Shutdown](https://www.youtube.com/watch?v=6rfBgphiCWM) (36m)

When a server must stop (deployment, scale-in, restart), it shouldn't "slam the door". A graceful shutdown lets it finish in-flight work, release resources and exit cleanly, avoiding lost requests, corrupted data and double charges.

---

## Contents

1. [The problem](#the-problem)
2. [Process lifecycle and signals](#process-lifecycle-and-signals)
3. [SIGTERM, SIGINT, SIGKILL](#sigterm-sigint-sigkill)
4. [Step 1: connection draining](#step-1-connection-draining)
5. [Step 2: resource cleanup](#step-2-resource-cleanup)
6. [The full sequence in code](#the-full-sequence-in-code)
7. [Summary](#summary)
8. [Interview questions](#interview-questions)

---

## The problem

[0:00](https://www.youtube.com/watch?v=6rfBgphiCWM&t=0s)

You're midway through a payment when the server restarts for a deployment. Zero-downtime deployments make sure the new server is ready before the old one stops, but eventually the **old server must stop**. At that moment:

- Does the payment get lost?
- Is the customer charged twice because of a race condition?

> **Graceful shutdown**: teach your backend good manners. Finish ongoing conversations, say goodbye, clean up, then close the door.

Benefits: better user experience, no data corruption, no double charges, no lost transactions or refund processing.

---

## Process lifecycle and signals

[3:35](https://www.youtube.com/watch?v=6rfBgphiCWM&t=215s)

Every application runs as a **process**, and every process has a lifecycle: born (starts), lives (executes), dies (terminates).

When the OS decides a process should stop, it doesn't just pull the plug. It follows a **protocol**, a conversation: *"Time to stop."* → *"Give me a few seconds."* That conversation happens through **signals**.

- Signals are a Unix mechanism for **IPC (inter-process communication)**.
- Servers overwhelmingly run on Linux (Unix-like), so signals are the relevant model.
- Your app **registers handlers** that wait for specific signals and run code when they arrive.

---

## SIGTERM, SIGINT, SIGKILL

[8:45](https://www.youtube.com/watch?v=6rfBgphiCWM&t=525s)

| Signal | Meaning | Who sends it | Catchable? |
|---|---|---|---|
| **SIGTERM** (terminate) | A **polite request**: "please finish up and leave." A gentle shoulder tap. Gives the app a window of a few seconds | Process managers, deployment/orchestration systems: **Kubernetes, systemd, PM2**, other programs | **Yes** |
| **SIGINT** (interrupt) | **User-initiated**. `Ctrl+C` in the terminal | Developers, mainly in development | **Yes** |
| **SIGKILL** (kill) | The **nuclear option**: like pulling the power plug | The OS, as a last resort | **No.** Can't be caught or ignored. The app gets **no** chance to clean up |

**Handle SIGINT and SIGTERM the same way.** Whether a human pressed Ctrl+C locally or PM2 sent SIGTERM in the cloud, the intent is identical: shut down cleanly.

> If you don't respect the polite signals, you'll eventually receive SIGKILL and lose the chance to clean up.

On receiving SIGTERM/SIGINT, the app must:

1. **Finish existing (in-flight) requests.**
2. **Clean up resources.**
3. **Exit.**

---

## Step 1: connection draining

[17:30](https://www.youtube.com/watch?v=6rfBgphiCWM&t=1050s)

**In-flight requests** = requests being processed right now (tens, hundreds, or thousands at once).

**Restaurant analogy** when closing time arrives:

1. Tell the host **not to let new customers in**.
2. Tell people already inside: "we're closing, you have 15–20 minutes to finish your meal."
3. Then close.

Same for your backend: **connection draining**.

```
1. STOP accepting new connections/requests.
2. LET existing in-flight requests finish.
3. THEN close.
```

How it looks per application type:

| App type | Draining means |
|---|---|
| **HTTP server** | Stop accepting new HTTP requests, finish active ones |
| **Database** | Finish existing queries/transactions, stop taking new ones, then close connections |
| **WebSocket** | **Notify clients** that you're closing, then close the sockets. Never abruptly |

### The timeout

Waiting forever isn't acceptable. Production systems set a **shutdown timeout** (commonly **30 seconds**, sometimes 60). Anything not finished by then is force-stopped.

- **Too short** → you cut off legitimate operations.
- **Too long** → slow deploys and a sluggish system.

Choose based on your typical request duration. WebSockets or long-running operations need more thinking.

Real systems also coordinate with **load balancers**, **health checks** and **service discovery** (deregistering the instance so no new traffic is routed to it).

---

## Step 2: resource cleanup

[26:17](https://www.youtube.com/watch?v=6rfBgphiCWM&t=1577s)

Like clearing your desk before leaving. "Resources" are anything the app acquired:

| Resource | Why clean up |
|---|---|
| **File handles** | Leaked handles consume memory and OS limits |
| **Network connections** | Operating systems cap open handles/connections per process |
| **Database connections / transactions** | Transactions must be **committed or rolled back explicitly**, otherwise they may be left inconsistent → deadlocks or data corruption |
| Temporary files, caches, other system resources | |

> **Release in the reverse order of acquisition.** If you opened Redis, then the DB, close the DB, then Redis. This avoids closing something that a later-acquired resource still depends on.

---

## The full sequence in code

[30:24](https://www.youtube.com/watch?v=6rfBgphiCWM&t=1824s)

A Go example (you don't need to read Go to follow it):

```text
1. Register a handler that waits for OS interrupt/terminate signals.
2. On signal → call shutdown():
     a. Shut down the HTTP server        (the framework stops accepting connections
                                           and finishes active ones)
     b. Close the database               (stops new queries, finishes current ones,
                                           releases the pool's TCP connections)
     c. Stop the background job server   (Asynq / Redis): "starting graceful shutdown,
                                           waiting for all workers to finish ... all workers
                                           have finished ... exiting"
3. Log "server exited properly".
```

Demo: start the server, press `Ctrl+C`, and the log shows DB closing, then the job server shutting down, then the clean exit message. It takes about a second since there were no in-flight requests.

Most frameworks (Node, Go, Rust, Python) provide ready-made shutdown helpers. What matters is **understanding what happens and why**.

```
Process lifecycle:   START ──► RUNNING ──► [signal received] ──► GRACEFUL SHUTDOWN ──► EXIT
```

---

## Summary

1. A server must stop at some point (deploy, scale, restart). Do it **gracefully** or risk lost/duplicated work and corrupted data.
2. The OS talks to a process through **signals**.
3. **SIGTERM** (from orchestrators) and **SIGINT** (Ctrl+C) are catchable and polite. Handle both **identically**. **SIGKILL** can't be caught.
4. Sequence: **capture signal → stop accepting new connections → finish in-flight requests (with a timeout, ~30 s) → release resources (reverse order) → exit.**
5. Resources: file handles, network and DB connections (commit/rollback transactions), background job workers, Redis.
6. Coordinate with load balancers and service discovery so no traffic is routed to a draining instance.

## Interview questions

**Q: What is graceful shutdown and why does it matter?**
Stopping an application in an orderly way: stop accepting new work, finish in-flight work, release resources, then exit. It prevents lost requests, double processing and data corruption during deploys.

**Q: SIGTERM vs SIGINT vs SIGKILL?**
SIGTERM: polite termination request (from orchestrators). SIGINT: user interrupt (Ctrl+C). SIGKILL: forced immediate kill that can't be caught or handled.

**Q: Why can't you handle SIGKILL?**
The kernel terminates the process directly. The signal never reaches user-space handlers.

**Q: What is connection draining?**
Stopping acceptance of new connections while letting existing in-flight requests complete before shutdown.

**Q: Why use a shutdown timeout?**
To bound how long a shutdown can take. Without it, a stuck request could block deploys indefinitely. When it expires, remaining work is forcibly terminated.

**Q: In what order do you release resources?**
Reverse of acquisition, so dependents are closed before the things they depend on.

**Q: What must happen to open DB transactions at shutdown?**
Commit or roll them back explicitly. Otherwise they may be left in an inconsistent state.

**Q: Why handle SIGINT and SIGTERM the same way?**
The intent (shut down cleanly) is the same whether a human or a process manager initiated it.

**Q: What else coordinates with graceful shutdown in a real deployment?**
Load balancer deregistration, health checks failing, service discovery updates, and for WebSockets, notifying clients first.
