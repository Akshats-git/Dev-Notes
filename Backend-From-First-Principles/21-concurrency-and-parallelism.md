# 21 — Concurrency and Parallelism: IO-Bound vs CPU-Bound

**Video:** [#22 Concurrency & Parallelism: IO Bound vs CPU Bound](https://www.youtube.com/watch?v=bs9MEYRTA30) (1h 28m)

A mental-model video, not a how-to. Why a server must do many things at once, what concurrency vs parallelism actually mean, why most backend work is **I/O-bound**, how **threads**, **event loops** and **Go's goroutines** implement concurrency, what `async/await` really is, and the race conditions you inherit.

> If you take only one thing away: understand **I/O-bound vs CPU-bound**.

---

## Contents

1. [The problem: a server that handles one request at a time](#the-problem-a-server-that-handles-one-request-at-a-time)
2. [Concurrency vs parallelism](#concurrency-vs-parallelism)
3. [I/O-bound vs CPU-bound](#io-bound-vs-cpu-bound)
4. [Model 1: threads](#model-1-threads)
5. [Model 2: event loop](#model-2-event-loop)
6. [Model 3: goroutines (virtual threads)](#model-3-goroutines-virtual-threads)
7. [What `async/await` really is](#what-asyncawait-really-is)
8. [Problems: race conditions](#problems-race-conditions)
9. [Which model for which workload](#which-model-for-which-workload)
10. [Summary](#summary)
11. [Interview questions](#interview-questions)

---

## The problem: a server that handles one request at a time

[0:00](https://www.youtube.com/watch?v=bs9MEYRTA30&t=0s)

If a web server could only process one request at a time, the other 999 users would wait or get "busy" errors. Impossible in production.

Think about a single request. It does a bit of work (routing, parsing JSON), then queries the database and **waits**:

| DB location | Typical wait for a simple query |
|---|---|
| Same machine (localhost) | 1–2 ms |
| Different availability zone | 20–30 ms |
| Distant region | 90–100 ms |

During that wait, a purely synchronous server's **CPU is idle**.

**How much is wasted?** A modern CPU executes roughly **3 billion instructions/second = 3 million per millisecond**. Waiting 100 ms means ~**300 million instructions you could have run, but ran zero**.

A typical API call may do 3–5 DB queries, a cache call and an email call: ~5 network operations at ~50 ms each = **250 ms waiting** vs. **~10 ms of actual CPU work**. The CPU and memory sit **idle ~95% of the time**. More than ~70% of a typical backend's time is I/O.

> Concurrency exists to use the CPU while waiting for I/O.

---

## Concurrency vs parallelism

[9:47](https://www.youtube.com/watch?v=bs9MEYRTA30&t=587s)

| | Parallelism | Concurrency |
|---|---|---|
| Meaning | Executing multiple instructions **at the same instant** | **Dealing with** multiple things at once, by structuring work so tasks can start, pause and resume |
| Needs | **Multiple CPU cores** (hardware) | Just one core is enough |
| One-liner | *Doing* many things at once | *Dealing with* many things at once |

One CPU core can only execute one instruction at one moment. Concurrency works on top of that:

```
1 core, concurrent:
time →   0       5          15            40      50      ...
CPU:   [ A: CPU ][ B: CPU    ][  (idle/other) ][ A: CPU ][ B: CPU ]
A:       work    -- waiting for DB response ----------->  work  done
B:                work    -- waiting for DB response ------------> work  done
```

When A starts waiting for I/O, the scheduler gives the CPU to B. Both requests are "in progress", but at any single instant only one is running. With **two cores** (parallelism), both could run at the same instant.

---

## I/O-bound vs CPU-bound

[18:32](https://www.youtube.com/watch?v=bs9MEYRTA30&t=1112s)

| | I/O-bound | CPU-bound |
|---|---|---|
| Waiting on | database, network, file system, logging, external APIs, stdin/stdout | nothing: actually computing |
| Examples | query the DB, call Stripe, write a log | image/video processing, matrix maths, encryption, hashing, JWT verification, heavy parsing |
| Best tool | **Concurrency** | **Parallelism** (multiple cores) |

- **Most backends are I/O-bound.** The bottleneck is I/O whether you have 1 or 8 cores.
- Small CPU work (validation, JSON parsing) takes 1–2 ms. It isn't heavy.
- A real backend needs **both**: concurrency for the many I/O waits, parallelism for heavy computation. But in typical SaaS backends, I/O dominates.

---

## Model 1: threads

[22:36](https://www.youtube.com/watch?v=bs9MEYRTA30&t=1356s)

Every concurrency feature in any language (goroutines, async/await, virtual threads) builds on **threads** or **event loops**.

A **thread** is an independent unit of execution provided by the **OS**. The OS gives each thread:

- a **stack** (function calls, local variables; ~8 MB virtual on Linux, physical memory allocated on use),
- an **instruction pointer** (where it is in the code).

A **scheduler** (controlled by the OS) decides which thread runs. It gives each a **time slice** (e.g. 2 ms), then pauses it (saving its state) and switches. This is **preemptive scheduling**: threads are stopped whether they like it or not.

**On a blocking (I/O) call:** the thread tells the OS it's blocked. The OS marks it *blocked*, switches to another thread, and when the I/O completes marks it *runnable* again.

**Sharing:** threads in the **same process** share memory (heap, globals), so they communicate by reading/writing the same data (fast, but dangerous, see race conditions). Threads in **different processes** are isolated.

With multiple cores, threads can truly run **in parallel**, which helps CPU-bound work.

### The costs of threads

| Cost | Detail |
|---|---|
| **Memory** | Each thread has its own stack. 10,000 threads at ~1 MB each is ~9 GB, so you run out of memory during a traffic spike |
| **Creation** | A syscall to the kernel: set up the stack, bookkeeping, add to scheduler: microseconds to milliseconds |
| **Context switching** | Save registers and state, pick next, restore: ~1–10 µs each. With thousands of threads this adds up to **ms of pure overhead** |

So "one OS thread per request" struggles at high concurrency.

*(Recommended reading mentioned: "Operating Systems: Three Easy Pieces".)*

---

## Model 2: event loop

[37:24](https://www.youtube.com/watch?v=bs9MEYRTA30&t=2244s)

Usually **one thread**. The core rule: **never block the event loop.**

- When a task needs I/O it **registers a callback** and gives control back to the loop: *"resume me when the DB responds, and meanwhile run other things."*
- The loop keeps a queue of tasks and, on every iteration, **checks whether any I/O has completed** (using OS facilities: **`epoll`** on Linux, **`kqueue`** on macOS, IOCP on Windows). If so, it runs the registered callback.

```
loop forever:
    check for completed I/O
    run the callbacks for the completed I/O
    run other ready tasks
```

**Pros for I/O-bound work:** one thread → **no context-switch overhead**, no big per-task stack memory. Very efficient.
**Trade-off:** a CPU-heavy callback (e.g. 100+ ms) **stalls everything**. The loop can't move on.

### Walkthrough: two requests

| | Threads | Event loop |
|---|---|---|
| Request A arrives | Parse JSON (CPU) → build query → send over TCP socket → `read()` **blocks the thread** → OS switches to another thread | Parse JSON → send query → **register callback** → return to the loop |
| Request B arrives | Same, on another thread | Same, loop picks it up |
| DB response arrives | OS wakes the thread, which continues (parse rows → native structure → JSON → respond) | `epoll` reports the socket readable → loop runs the callback |
| Resume mechanism | OS thread stacks and instruction pointers (heavy) | **Callbacks** (light) |

Threads code (Python-like) hides the blocking:

```python
def handle_request(req):
    user = db.query("SELECT * FROM users WHERE id = %s", req.id)   # thread blocks here
    return json(user)
```

Event-loop style (Node), old callback form:

```js
db.query("SELECT * FROM users WHERE id = ?", [id], function (err, user) {
  sendResponse(user);
});
```

---

## Model 3: goroutines (virtual threads)

[1:00:24](https://www.youtube.com/watch?v=bs9MEYRTA30&t=3624s)

Go's approach: **goroutines**, lightweight "virtual threads" managed by the **Go runtime scheduler**, not the OS.

- Go's HTTP server spawns **a new goroutine per request** (the `go` keyword; you can find this in the stdlib's `Serve` loop). That's affordable because goroutines are cheap.
- The runtime runs a small number of **OS threads** (by default one per CPU core, `GOMAXPROCS`), each acting like a queue of goroutines.

```
Go runtime scheduler
   goroutines  G1 G2 G3 G4 ... G100000
        ▼ (mapped onto)
   OS threads  M1  M2  M3  M4        (≈ CPU cores)
        ▼
   CPU cores
```

- When a goroutine hits blocking I/O (`db.Query`), the Go scheduler **pauses it and runs another goroutine on that same OS thread**. Switching is mostly a **pointer switch**, and very cheap.
- You can have 100× more goroutines than OS threads. The code **reads like blocking, sequential code**, but behaves concurrently.

This is why Go is efficient at both I/O and CPU work, without an event loop.

---

## What `async/await` really is

[1:10:38](https://www.youtube.com/watch?v=bs9MEYRTA30&t=4238s)

`async/await` is **syntactic sugar over callbacks**. Callbacks were hard to read (nested "callback hell"). `await` says: *"hand control back to the event loop; when this I/O completes, resume from here."* Everything after the `await` in that function is conceptually a callback.

```js
async function fetchUserData(userId) {
  const user   = await db.getUser(userId);      // give up control until done
  const orders = await db.getOrders(user.id);   // again
  return { user, orders };
}
```

The runtime compiles this into a **state machine**:

```js
function fetchUserData(userId) {
  let state = 0, user, orders;
  return function step() {
    switch (state) {
      case 0: state = 1; db.getUser(userId).then(r => { user = r; step(); }); return;
      case 1: state = 2; db.getOrders(user.id).then(r => { orders = r; step(); }); return;
      case 2: return { user, orders };
    }
  };
}
```

Each `await` is a state transition. That explains two things:

1. **You can only use `await` inside an `async` function**, because the function has to be turned into a state machine.
2. **Blocking the event loop is catastrophic.** The state machine can't advance, and *nothing* runs.

---

## Problems: race conditions

[1:16:13](https://www.youtube.com/watch?v=bs9MEYRTA30&t=4573s)

Almost all concurrency problems come from **shared state**.

### Lost update (threads)

`counter++` is three steps: **read → add → write**. Two threads interleave:

| ms | Thread A | Thread B | counter |
|---|---|---|---|
| 1 | reads 0 | | 0 |
| 2 | | reads 0 | 0 |
| 3 | adds 1 (reg=1) | | 0 |
| 4 | | adds 1 (reg=1) | 0 |
| 5 | writes 1 | | 1 |
| 6 | | writes 1 | **1 (should be 2)** |

This is a **race condition** (lost update).

### Even single-threaded `async/await` isn't safe

```js
let balance = 100;
async function withdraw(amount) {
  if (balance >= amount) {          // check
    await processWithdrawal(amount); // gives up control here!
    balance -= amount;               // act
  }
}
// call withdraw(100) twice "at once"
```

Both calls pass the `balance >= 100` check before either deducts (control is yielded at the `await`). The balance ends at **−100**.

### Fixes

| Technique | Idea |
|---|---|
| **Locks / mutexes** | Mutual exclusion: only one thread may be in the critical section. Others wait |
| **Channels** (Go) | Don't share the variable. Goroutines send **messages**, and one owner goroutine updates it |
| Database-level safety | Transactions, atomic updates (`UPDATE ... SET balance = balance - $1 WHERE balance >= $1`) |

```python
import threading
lock = threading.Lock()
def increment():
    global counter
    with lock:            # only one thread inside at a time
        counter += 1
```

(The speaker deliberately keeps it brief: the aim is the I/O vs CPU mental model.)

---

## Which model for which workload

[1:26:00](https://www.youtube.com/watch?v=bs9MEYRTA30&t=5160s)

| Workload | Best fit |
|---|---|
| **I/O-bound, high concurrency** (web servers, API gateways, services calling other services) | **Event loop / `async`/`await`**, **goroutines**, Java **virtual threads** |
| **CPU-bound** (number crunching, image/video, ML, hashing) | **Threads with parallelism** across cores |
| Typical backend | Mostly I/O → lean on **concurrency**; add parallelism for heavy computation |

---

## Summary

1. Servers must handle many requests at once, since otherwise CPUs sit idle ~95% of the time waiting on I/O.
2. **Concurrency** = structuring work so tasks interleave (works on 1 core). **Parallelism** = executing at the same instant (needs multiple cores).
3. **I/O-bound** work (DB, network, files, logs) benefits from concurrency. **CPU-bound** work (compute) benefits from parallelism. Most backend work is I/O-bound.
4. **Threads**: OS-managed, preemptive, share memory. Costly: stack memory, creation, context switches.
5. **Event loop**: one thread, callbacks, OS notification (`epoll`/`kqueue`). Efficient for I/O, but **never block the loop**.
6. **Goroutines**: cheap virtual threads scheduled by the Go runtime on a few OS threads. One goroutine per request.
7. **`async/await`** = sugar over callbacks, compiled to a **state machine**.
8. Shared state causes **race conditions**, even with single-threaded `async`. Use locks, channels, or database transactions/atomic updates.

## Interview questions

**Q: Concurrency vs parallelism?**
Concurrency is dealing with multiple tasks by interleaving them (possible on one core). Parallelism is executing multiple tasks simultaneously on multiple cores.

**Q: Why are most backends I/O-bound, and what does that imply?**
Requests spend most of their time waiting on databases, networks and files. So concurrency matters more than raw parallelism.

**Q: What is the cost of threads?**
Memory per stack, creation syscalls, and context-switch overhead, all of which hurt at thousands of threads.

**Q: How does an event loop work?**
One thread loops: it checks which I/O operations have completed (via `epoll`/`kqueue`) and runs the registered callbacks for them.

**Q: What does it mean to "block the event loop" and why is it bad?**
Running long CPU work (or a synchronous blocking call) in the loop's thread. Nothing else can run until it finishes, so all requests stall.

**Q: What is a goroutine and how is it different from an OS thread?**
A lightweight task scheduled by the Go runtime onto a small pool of OS threads. Cheaper to create and switch, so you can have one per request.

**Q: What does `await` do?**
Suspends the current async function and returns control to the event loop until the awaited operation completes, then resumes. The compiler turns the function into a state machine.

**Q: Can you have race conditions in single-threaded JavaScript?**
Yes. Any `await` between a check and an update yields control, letting another task change the state, e.g. a double-withdraw bug.

**Q: How do you prevent race conditions?**
Locks/mutexes, message passing (channels), or atomic/transactional operations at the database level.

**Q: Which model for heavy image processing vs an API gateway?**
Image processing is CPU-bound: threads/processes across cores. An API gateway is I/O-bound: event loop or goroutines.
