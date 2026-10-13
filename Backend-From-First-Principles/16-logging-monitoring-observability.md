# 16 — Logging, Monitoring and Observability

**Video:** [#18 Logging, Monitoring and Observability](https://www.youtube.com/watch?v=5PEuwgLOQQM) (40m)

Three related practices for understanding what a distributed backend is doing. The speaker stresses that these are **practices on a spectrum**, not a checklist: no company does all of it perfectly. This is the one video of the series that shows code (a Go to-do app instrumented with New Relic), because the practice only makes sense next to code.

(A sponsor segment at ~14:25–18:00 is skipped here.)

---

## Contents

1. [Why we need them](#why-we-need-them)
2. [The three concepts](#the-three-concepts)
3. [How they work together](#how-they-work-together)
4. [Logging in practice](#logging-in-practice)
5. [Instrumentation and OpenTelemetry](#instrumentation-and-opentelemetry)
6. [The demo: a Go app with New Relic](#the-demo-a-go-app-with-new-relic)
7. [Tools](#tools)
8. [Summary](#summary)
9. [Interview questions](#interview-questions)

---

## Why we need them

[0:00](https://www.youtube.com/watch?v=5PEuwgLOQQM&t=0s)

Modern backends run on many servers, in many regions, serving users worldwide. You need practices and tools to **keep track of what's happening** across all services and infrastructure.

---

## The three concepts

[2:33](https://www.youtube.com/watch?v=5PEuwgLOQQM&t=153s)

| Concept | What it is | Answers |
|---|---|---|
| **Logging** | **Recording** important events (including suspicious and security-related ones) with **metadata**: user ID, latency, which function ran. A journal of your app's life | **What happened, and why?** |
| **Monitoring** | Continuously **tracking the state** of the app and its parts (server CPU, memory, requests/second, open DB connections), as near-real-time data (typically 10–15 s delay) and aggregated over time | **How healthy is it now, and what are the trends?** |
| **Observability** | The ability to determine a system's **internal state from its external outputs**. A system is observable only if it has all **three pillars**: **logs, metrics, traces** | **Where exactly is the problem?** |

### The three pillars

| Pillar | Meaning |
|---|---|
| **Logs** | Records of events across the request lifecycle and app start/stop |
| **Metrics** | Concrete **numbers** over time: requests processed, requests failed (status > 200-ish), to-dos created, error rate, latency, throughput. Pre-decide which ones you care about |
| **Traces** | A **transaction** that follows one request across every component it touches: load balancer → handler → validation → service → repository → database. Shows where it started and where it failed |

### Monitoring vs observability

A decade ago, error prevention relied mostly on **monitoring**, which tells you **that** something is wrong (set up alerts in Grafana, say). **Observability** tells you that **and what exactly is wrong**, provided you implemented logs, metrics and traces.

---

## How they work together

[10:14](https://www.youtube.com/watch?v=5PEuwgLOQQM&t=614s)

A real incident flow:

```
1. ALERT: error rate > 80%  →  Slack message (via webhook): "API service is unhealthy"
2. METRICS dashboard: confirms the error rate, shows throughput, latency...
3. LOGS: click through to the failing log lines for that metric (all the failed requests)
4. TRACE: click a 500 log → see the request path
          e.g. handler → service → repository → failed HERE
5. Fix.
```

Metrics show **patterns and trends**, logs show **what happened**, traces show **how components interacted**.

---

## Logging in practice

[18:31](https://www.youtube.com/watch?v=5PEuwgLOQQM&t=1111s)

### Log levels

| Level | Use | In production? |
|---|---|---|
| **DEBUG** | Detailed troubleshooting info during development | Usually **disabled** (too noisy) |
| **INFO** | General operations and business events. A to-do was created. Any successful operation | Yes |
| **WARN** | Between info and error. Not a success, not critical, not our fault. E.g. a user typed a wrong password | Yes |
| **ERROR** | Something failed: validation error, DB query failure. The most common reason for logging | Yes |
| **FATAL** | Very serious. The app is shutting down (and will restart per infra config) | Yes |

The log level is itself configuration: `debug` in local/dev, `info` in production.

### Structured vs unstructured

[21:10](https://www.youtube.com/watch?v=5PEuwgLOQQM&t=1270s)

| | Console / plain text | Structured (JSON) |
|---|---|---|
| For | **Development** | **Production** |
| Why | Human-readable, colourful, easy to scan | Log pipelines (ELK, Loki/Promtail/Grafana) can **parse** it and pull out fields such as `user_id` and `request_id`. Parsing free text is hard and error-prone |

### What to log

Every important event, with context: timestamp, user ID, request ID, DB query/operation, error details, method, route, span ID. (Never log secrets or PII. See [14](14-error-handling-and-fault-tolerance.md).)

---

## Instrumentation and OpenTelemetry

[28:20](https://www.youtube.com/watch?v=5PEuwgLOQQM&t=1700s)

- **Instrumentation** = measuring attributes of your code so the system can be observed. A middleware that wraps every request is a common way to instrument.
- **OpenTelemetry (OTel)** = an open standard (APIs, SDKs, best practices) for instrumenting apps in any major language. Even with a proprietary tool like New Relic, you can plug in an OTel collector for more control.

---

## The demo: a Go app with New Relic

[23:42](https://www.youtube.com/watch?v=5PEuwgLOQQM&t=1422s)

Don't try to memorise the code. The point is the **practice**.

1. **Logger setup before the app starts.**
   - `getLogLevel()` → `debug` in dev, `info` in production.
   - Format: `console` (readable) in dev, `json` in production. Same logs, shown both ways. JSON is unreadable to humans but easy for tools.

2. **A New Relic middleware** wraps the router and instruments every request.

3. **A tracing middleware** (the first point of contact for a request) creates a **transaction** with attributes: service name, environment, IP, user agent, request ID, user ID, email, tenant ID. It stores the transaction in the **request context** (see [08](08-handlers-services-repositories-middleware-context.md)).

4. **In the service method** (`createTodo`):
   - Pull the transaction from the context. `defer` ending that segment.
   - Add attributes (user ID, title, priority).
   - **Log INFO**: "initiating create todo".
   - Run the DB operation. On error → **log ERROR**, attach the error to the trace, mark the operation as failed.
   - On success → add attributes, **log DEBUG** ("created successfully", hidden in production) and a **business-event log** (todo ID, title, category, priority).

   One request thus produces **one trace** spanning middleware → validation → service.

5. **Dashboard** (production mode, JSON logs on): hit the API without a token → `401 Unauthorized` errors appear under **Errors → HTTP**, with their related **logs** (app name, environment, error code, host, IP, level, message, method, route, span ID, timestamp) linked to a **trace**. **Metrics** show average transaction time, throughput and error rate. The **Go runtime** page shows GC time and memory use (~3 MB).

---

## Tools

[38:08](https://www.youtube.com/watch?v=5PEuwgLOQQM&t=2288s)

| Route | Tools |
|---|---|
| **Open source** | **Grafana** (dashboards), **Prometheus** (metrics), **Loki** + **Promtail** (logs), **Jaeger** (traces), ELK stack. Common in enterprises, but takes a team to maintain |
| **Proprietary / all-in-one** | **New Relic**, **Datadog**. Simpler integration if you lack the team or time |

> Making this work is a **collective effort**: developers instrument and log at code level, DevOps sets up collection and monitoring infrastructure.

---

## Summary

1. **Logging** = what happened. **Monitoring** = current health and trends. **Observability** = infer the internal state from outputs, via **logs + metrics + traces**.
2. Monitoring says *something's wrong*. Observability helps find *what and where*.
3. Typical flow: **alert → metrics → logs → trace**.
4. **Levels**: DEBUG, INFO, WARN, ERROR, FATAL. Debug off in prod.
5. **Dev**: console logs. **Prod**: structured JSON logs.
6. **Instrument** requests with a middleware that creates a transaction/trace, and put it in the request context.
7. **OpenTelemetry** is the open standard for instrumentation.
8. Tools: Grafana/Prometheus/Loki/Jaeger (open source), New Relic/Datadog (proprietary).
9. These practices exist on a **spectrum**. Do what's useful and sustainable.

## Interview questions

**Q: Logging vs monitoring vs observability?**
Logging records events. Monitoring tracks system state and trends in near-real time. Observability lets you infer internal state from outputs, built from logs, metrics and traces.

**Q: What are the three pillars of observability?**
Logs, metrics, traces.

**Q: What is a trace?**
The end-to-end record of one request as it passes through every component, showing timing and where it failed.

**Q: Which log levels would you use for: wrong password / DB query failure / successful to-do creation / app crash?**
WARN / ERROR / INFO / FATAL.

**Q: Why JSON logs in production but plain text in dev?**
JSON is machine-parseable, so log tools can index fields (user ID, request ID). Plain text is easier for humans to read locally.

**Q: How do you debug a spike in errors?**
Alert triggers → check metrics to confirm scope → drill into related logs → open a trace to find the failing component.

**Q: What is OpenTelemetry?**
A vendor-neutral standard and set of SDKs for instrumenting applications to produce traces, metrics and logs.

**Q: Why use request/trace IDs in logs?**
To correlate every log line and span belonging to a single request across services.

**Q: Build or buy for observability?**
Open source (Grafana stack) gives control but needs a team to run it. Managed tools (New Relic, Datadog) trade cost for simplicity.
