# 12 — Task Queues and Background Jobs

**Video:** [#14 Task queues and background jobs](https://www.youtube.com/watch?v=r-nQsyguU1Y) (56m)

Why work should leave the request/response cycle, how a task queue works (producer → broker → consumer), what happens on failure (retries, acknowledgements, visibility timeout), the kinds of tasks you'll build, and the design considerations and best practices.

---

## Contents

1. [What a background task is](#what-a-background-task-is)
2. [The motivating example: signup email](#the-motivating-example-signup-email)
3. [The asynchronous version](#the-asynchronous-version)
4. [Retries and failure](#retries-and-failure)
5. [What to offload](#what-to-offload)
6. [How a task queue works](#how-a-task-queue-works)
7. [Acknowledgements and visibility timeout](#acknowledgements-and-visibility-timeout)
8. [Types of tasks](#types-of-tasks)
9. [Design considerations](#design-considerations)
10. [Best practices](#best-practices)
11. [Summary](#summary)
12. [Interview questions](#interview-questions)

---

## What a background task is

[0:00](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=0s)

> A **background task** is any code that runs **outside the request-response lifecycle**.

It doesn't have to complete immediately for the client to get its answer, so it isn't mission-critical to the response. That means it can be offloaded to a separate process and finished on its own schedule.

---

## The motivating example: signup email

[1:35](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=95s)

A user signs up. The backend validates, stores the user, creates a verification code, and then must **send a verification email** (a link, or a 6–8 digit one-time code).

Sending an email means building an HTML template, then calling a **third-party email provider's API** (Resend, Mailgun, Brevo, ...) with recipient, subject, sender and content. That is **an HTTP call to a server we don't control**.

### Doing it synchronously (the problem)

```
Client ──signup──► Server: validate → save user → create code → call email API ──► provider
                                                                  │ provider is down / slow
                                                                  ▼
                                                      the whole signup fails (500)
```

Two bad outcomes:

1. **No error handling:** the whole signup API fails. Terrible UX.
2. **With error handling:** signup succeeds and the UI says "we've sent you an email", but the email never went out. The user has to click "resend", hoping the provider is back.

---

## The asynchronous version

[9:17](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=557s)

```
Client ──signup──► Server (PRODUCER)
                     │ validate → save user → create code
                     │ serialize "send email" task (JSON) ──► QUEUE (broker)
                     │ return 200/201 immediately
                     ▼
              UI: "We've sent a verification email."

                                     QUEUE ──► CONSUMER / WORKER (separate process)
                                                 deserialize task → handler:
                                                 call email provider API → email sent
```

- The **producer** (your app) creates the task, serializes all the data the worker needs, pushes it onto the queue, and **returns right away**.
- A **consumer/worker**, running in a **different process** (maybe a different code base or node), takes the task, deserializes it into the language's native type (dict / object / struct), and runs the **registered handler**: the same code you'd have run synchronously.
- You can have separate queues for different jobs (emails, in-app notifications, mobile push) with different consumers.
- Latency is usually milliseconds to a few seconds, fine for a verification email that is valid for 15–20 minutes.

---

## Retries and failure

[16:34](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=994s)

If the email provider is down, the **worker's handler fails**, not the user's request. The task goes back into the queue to be **retried**.

**Exponential backoff:** each retry waits longer.

```
fail → retry after 1 min → fail → 2 min → fail → 4 min → fail → 8 min ...   (up to a max, e.g. 5 retries)
```

Provider outages usually last seconds, so the 2nd or 3rd try typically succeeds. The user still gets their email.

**Libraries:** Celery (Python), BullMQ (Node.js), Asynq (Go).

**Benefits:** a responsive API (not blocked by slow/external work), no timeouts from external dependencies, **built-in retry and failure detection**, better UX.

---

## What to offload

[20:38](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=1238s)

| Task | Why it's background work |
|---|---|
| **Sending emails** | Depends on an external service |
| **Processing images / videos** | Resize/encode to many sizes and formats for different devices and networks. CPU heavy |
| **Generating reports** | e.g. a project-management app sends daily/weekly/monthly PDF/HTML reports at midnight or Sunday. Scheduled (cron-style) |
| **Push notifications** | Backend stores the device token and calls Google's/Apple's push service, which actually delivers it. Another external call |

Task-queue libraries (Celery, BullMQ) also support **scheduled** tasks at set times/intervals.

---

## How a task queue works

[25:14](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=1514s)

> A task queue is a system for **managing and distributing background jobs**: a to-do list for your backend.

| Component | Role |
|---|---|
| **Producer** | Your application code. Creates a task containing all the info the worker needs (user ID, name, email, image, report data...), **serializes** it (JSON) and **enqueues** it |
| **Queue / Broker** | Stores tasks until a worker is ready. A temporary holding area |
| **Consumer / Worker** | A **separate process** (or thread). Monitors the queue, **dequeues** a task, deserializes it and runs the handler |
| **Result backend** (optional) | Stores results/status |

**Enqueue** = push onto the queue. **Dequeue** = take off.

**Broker technologies:** RabbitMQ, **Redis** (pub/sub, lists, or streams), **Amazon SQS** (managed, multi-region, good for scaling across the world).

---

## Acknowledgements and visibility timeout

[31:24](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=1884s)

- When a worker finishes, it sends an **acknowledgement (ack)** to the queue. The queue then **removes** the task.
- If there's no ack, the queue decides whether to apply retry mechanisms.

### Visibility timeout

A task a worker is currently processing is "in progress" and hidden from other workers for a period: the **visibility timeout**. If the worker **doesn't ack within that timeout** (it crashed, or an external service hung), the queue **makes the task visible again** for another worker. This way, a task isn't lost just because a worker died after dequeuing it.

Frameworks like Celery, BullMQ and Asynq handle these edge cases for you.

---

## Types of tasks

[34:29](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=2069s)

| Type | Description | Examples |
|---|---|---|
| **One-off** | A single trigger at a specific moment. Most common. | verification email, welcome email, password-reset email, "someone texted you" notification |
| **Recurring** | Executed periodically | daily/monthly/annual reports; **cleanup / maintenance** (e.g. delete orphaned sessions every month in a stateful-auth system) |
| **Chain** | Parent-child dependency. A task starts only after its parent succeeds | LMS video upload (below) |
| **Batch** | One task fans out to many, or many identical tasks run at once | "Delete account"; sending thousands of weekly reports at once |

### Chain example: LMS video upload

[37:36](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=2256s)

```
upload video (e.g. to S3 via pre-signed URL)
        │
   encode video (multiple resolutions/formats)
        ├───────────────────────────┐
  generate thumbnails        generate audio transcription / subtitles    ← run in PARALLEL
        │
  process thumbnails (resize variants)
```

Thumbnail generation and transcription both depend on encoding, but not on each other, so they run in parallel. Thumbnail *processing* depends on thumbnail *generation*.

### Batch example: delete account

[41:11](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=2471s)

You can't delete all of a user's data (across tables, shards, regions) inside one request: it might take a minute or more. Instead:

1. `DELETE /account` enqueues a "delete account" task and immediately returns `200`. The UI logs the user out.
2. Optionally, a **grace period** (3 or 7 days) lets the user log back in and cancel.
3. The worker deletes the user's projects, entities, profile, assets (logos, cover images), the account itself, and sends a confirmation email. It can itself enqueue many sub-tasks.

---

## Design considerations

[45:50](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=2750s)

| Concern | What to do |
|---|---|
| **Idempotency** | Design tasks so running them **multiple times** (retries!) has no bad side effects. E.g. do the whole delete in **one transaction**; on failure, roll back so a retry starts clean from 0%. |
| **Error handling** | Be thorough. The code runs in a separate process where it's easy to miss a case. Catch, **log**, and let the queue retry |
| **Monitoring** | Track queue length, successes, failures, and failure reasons (external service vs internal bug). Use metrics with tools like Prometheus + Grafana |
| **Scalability** | Design so you can **add more consumers horizontally** when load grows |
| **Ordering** | If order matters, make sure your library/broker supports ordered delivery |
| **Rate limiting** | If tasks call external services, rate-limit them to avoid overloading, exceeding quotas, or running up bills |

---

## Best practices

[51:33](https://www.youtube.com/watch?v=r-nQsyguU1Y&t=3093s)

1. **Keep tasks small and focused.** One processing unit per task. If one fails, others aren't affected. Use **chains** for dependencies. A child can retry on its own, while a monolithic task would redo everything.
2. **Avoid long-running tasks.** Break them into smaller chunks (parallel tasks or parent-child).
3. **Proper error handling and logging.** So the queue can retry and you can debug.
4. **Monitor queue length and worker health** with alerts. If the queue grows past a limit, scale. If workers die, find out why.

---

## Summary

1. Background tasks run **outside the request-response cycle** so APIs stay fast and don't depend on slow or external services.
2. **Producer** enqueues a serialized task, **broker/queue** holds it, **consumer/worker** (separate process) dequeues and executes it.
3. Failures are retried automatically, typically with **exponential backoff** and a max retry count.
4. Workers **ack** on success. A **visibility timeout** re-offers unacknowledged tasks so nothing is lost.
5. Task kinds: **one-off, recurring (scheduled), chain, batch**.
6. Make tasks **idempotent**, observable, scalable (add consumers), and rate-limited when calling external APIs.
7. Keep tasks **small**, **focused**, **short**.
8. Tools: Celery / BullMQ / Asynq; brokers: RabbitMQ / Redis / SQS.

## Interview questions

**Q: Why not just send the email inside the signup request?**
The email provider is an external dependency. If it's slow or down, your signup fails or silently drops the email. Offloading decouples your API's availability from theirs and enables retries.

**Q: Explain producer, broker, consumer.**
The producer creates and enqueues a task. The broker (queue) stores it. The consumer, a separate process, dequeues and executes it.

**Q: What is exponential backoff and why?**
Increase the delay between retries (1 min, 2, 4, 8...). It avoids hammering a recovering service and lets transient failures clear.

**Q: What's a visibility timeout?**
The period during which a dequeued task is considered "in progress". If no ack arrives in time, the task becomes visible again so another worker can take it, so a crash doesn't lose work.

**Q: Why must tasks be idempotent?**
Retries and redelivery mean a task can run more than once. Running it again must not duplicate side effects (double charge, duplicate email, half-deleted data).

**Q: Chain vs batch tasks?**
Chain: tasks depend on each other (child starts after the parent succeeds). Batch: many tasks (or sub-tasks) run together, e.g. delete-account or mass reports.

**Q: Give examples of recurring tasks.**
Scheduled reports, database backups, cleaning abandoned sessions, cache/log cleanup.

**Q: Which brokers and libraries exist?**
Brokers: RabbitMQ, Redis, Amazon SQS. Libraries: Celery (Python), BullMQ (Node), Asynq (Go).

**Q: How would you scale a task-processing system?**
Add more consumer instances horizontally, monitor queue length, and use a managed/distributed broker such as SQS when needed.

**Q: How do you handle an "account deletion" that takes too long for a request?**
Return success immediately, enqueue a delete-account task, optionally add a grace period, and let a worker do the multi-step deletion (ideally idempotent, transactional).
