# 02 — What Is a Backend, How It Works, and Why We Need One

**Video:** [#3 What is a Backend, how do they work and why do we need them?](https://www.youtube.com/watch?v=6Ss4dJD9Kzg) (19m)

This lecture traces one real request across the internet (browser → DNS → firewall → server → reverse proxy → app), then answers the obvious question: why can't the frontend just do all of this itself?

---

## Contents

1. [A working definition](#a-working-definition)
2. [The journey of a request](#the-journey-of-a-request)
3. [Why a backend exists: data](#why-a-backend-exists-data)
4. [How a frontend really works](#how-a-frontend-really-works)
5. [Why we can't put backend logic in the frontend](#why-we-cant-put-backend-logic-in-the-frontend)
6. [Summary](#summary)
7. [Interview questions](#interview-questions)

---

## A working definition

[0:00](https://www.youtube.com/watch?v=6Ss4dJD9Kzg&t=0s)

> A backend is a computer **listening for requests** (HTTP, WebSocket, gRPC, ...) on an **open port** (80, 443, ...), reachable over the internet, so that clients can send it data or receive data from it.

It's called a **server** because it *serves* content: static files (images, JS, HTML) or dynamic data like JSON. It also accepts data the client sends.

That is the textbook version. The rest of the lecture makes it physical.

---

## The journey of a request

[0:30](https://www.youtube.com/watch?v=6Ss4dJD9Kzg&t=30s)

The demo is a Node server on an AWS EC2 instance at a subdomain like `backend-demo.example.xyz`, serving `GET /users`. Open the browser dev tools Network tab (with "disable cache" on) and you can see the request leave the browser and the JSON response return. Now trace every hop.

```
Browser
   │  1. resolve domain name
   ▼
DNS server  ──►  A record: backend-demo → 3.x.x.x (EC2 public IP)
   │
   ▼
AWS firewall (Security Group)   ← only allowed ports pass
   │  ports 22 (SSH), 80 (HTTP), 443 (HTTPS)
   ▼
EC2 instance
   │
   ▼
Nginx (reverse proxy)  listens on 80/443
   │  server_name backend-demo.example.xyz → proxy_pass localhost:3001
   ▼
Node server on localhost:3001   (managed by pm2)
```

### Hop 1 — DNS

[1:31](https://www.youtube.com/watch?v=6Ss4dJD9Kzg&t=91s)

The browser sees a **domain name**, not an IP. DNS translates one into the other. Two record types matter here:

| Record | Points to |
|---|---|
| **A record** | An IP address |
| **CNAME record** | Another domain or subdomain |

The subdomain `backend-demo` has an **A record** pointing to the public IP of the EC2 instance. (DNS is a huge topic, this is the oversimplified view.)

### Hop 2 — The firewall

[3:05](https://www.youtube.com/watch?v=6Ss4dJD9Kzg&t=185s)

Before traffic reaches the machine it passes AWS's firewall, the **Security Group**. It declares which ports are reachable from the internet. Three are open here:

| Port | Why |
|---|---|
| 22 | SSH, to log into the instance's terminal |
| 80 | HTTP |
| 443 | HTTPS |

> If 80 and 443 aren't allowed, AWS blocks the request right there and it never reaches your server.

### Hop 3 — The reverse proxy (Nginx)

[4:08](https://www.youtube.com/watch?v=6Ss4dJD9Kzg&t=248s)

A **reverse proxy** is a server in front of other servers, so you manage redirects and config in one place instead of in every app. Nginx is the common choice.

What the Nginx config does in the demo:

- Certbot manages SSL certificates automatically (not the focus).
- It listens on **port 80** and redirects to **port 443** (HTTPS).
- A `server_name` block says: for requests to *this* domain, forward to `localhost:3001`, which is where the Node server is listening.

### Hop 4 — The application server

[5:39](https://www.youtube.com/watch?v=6Ss4dJD9Kzg&t=339s)

`pm2 list` shows two processes (frontend and backend). The backend is a Node server on `localhost:3001`. From the instance's point of view, the app is just a local process. `curl localhost:3001/users` returns the same JSON. **Nginx plus DNS is what exposes a local process to the internet.**

This is exactly what you do in development too: open `localhost:3000/users` and get the same response.

### Whole path

```
browser → DNS → AWS (firewall) → EC2 → Nginx → localhost:3001 (app)
```

---

## Why a backend exists: data

[7:13](https://www.youtube.com/watch?v=6Ss4dJD9Kzg&t=433s)

Think about liking a friend's post on Instagram. Between your tap and their notification:

1. The app sends a request to the server.
2. The server parses it and works out **who you are**.
3. It **persists** the action (the like) in a database.
4. It finds the **author of the post** and triggers a notification to them.

For all of that to work there must be a **centralised computer holding the information of all users**. Your feed is personalised to who you follow. Your friend only gets notifications meant for them. Someone has to hold all that state.

> Strip the backend down to one word: **data**. Fetching data, receiving data, persisting data, and every action that deals with data.

---

## How a frontend really works

[9:45](https://www.youtube.com/watch?v=6Ss4dJD9Kzg&t=585s)

"Why not do everything in the frontend? It's also a computer. Connect to the DB from there, spread the work around, get better performance."

To answer that, trace a frontend request (the demo is a Next.js app on the same EC2 box):

1. The first request returns an **HTML file**.
2. The browser then fetches every other resource the HTML references: JS, CSS, images, fonts, each as a separate request.
3. Same DNS → firewall → Nginx path. Only the Nginx config differs: this domain proxies to `localhost:3000` instead of `3001`.
4. Once the **CSS** arrives, the browser **paints** the page.
5. Once the **JS** arrives, it **hydrates**: attaches event listeners, so buttons and links start working.

### The key difference

| | Backend | Frontend |
|---|---|---|
| Where code runs | **On the server**. You send a request and get a result | **In the user's browser**. The server just sends the code |
| Runtime | Server's OS (Node, Go, ...) | The browser |

The browser is a **sandbox**: isolated from the OS, processes and file system. JS can only touch:

- the DOM
- browser APIs (localStorage, cookies, ...)
- external APIs, **but only if** those APIs send the right CORS headers

**CORS** is a browser security policy: JS loaded from `frontend.example.xyz` may only call the *same origin* unless the other server explicitly allows it via HTTP headers. (Later video.)

This sandbox is deliberate. The browser downloads code from a stranger's server and runs it on your machine. Without isolation, any website could read your files and send them away.

---

## Why we can't put backend logic in the frontend

[15:26](https://www.youtube.com/watch?v=6Ss4dJD9Kzg&t=926s)

| # | Reason | Detail |
|---|---|---|
| 1 | **Security restrictions** | Backends often need the file system (write logs, read env vars). Browsers forbid that. |
| 2 | **No free access to external APIs** | Backends call many third-party servers. In a browser, each one must send the right CORS headers, and you don't control third parties. |
| 3 | **Databases** | Server runtimes have native DB drivers (`pg` for Postgres, the MongoDB driver). They use sockets, binary data and **persistent connections**. Browsers can't. Servers also keep a **connection pool**, reused across thousands of requests per second. If every browser opened its own DB connection, the DB server would be overwhelmed. |
| 4 | **Computing power** | Clients vary hugely (a 256 MB single-core device vs. a laptop). Heavy business logic would lag or break on weak devices. A central server can be given more CPU and memory when needed. |

The list can go on, but the point is clear. And those are only the technical reasons. The earlier requirement for one central, trusted holder of everyone's data is enough on its own.

---

## Summary

1. A backend is a computer **listening on an open port** and serving/accepting data.
2. A request goes **browser → DNS (A record → IP) → firewall (open ports 80/443) → server → Nginx reverse proxy → app on localhost:port**.
3. A **reverse proxy** centralises redirects, TLS and config in front of one or more apps.
4. The backend exists to hold **centralised data** and run actions on it.
5. A frontend's code is **downloaded and run in the browser**, a **sandbox** with no file system, no raw sockets, and CORS limits.
6. That's why DB access, secrets, third-party calls and heavy work belong on the **server**.

## Interview questions

**Q: What happens when you type a URL and hit enter (backend side)?**
DNS resolves the domain to an IP (A record). The request goes through the cloud firewall (needs 80/443 open) to the machine. A reverse proxy such as Nginx matches the host and forwards to the local app port. The app processes the request and the response retraces the same path.

**Q: What is a reverse proxy and why use one?**
A server in front of backend servers. It terminates TLS, routes by domain/path, and lets you change config in one place without touching each app.

**Q: A frontend is also a computer. Why can't it connect directly to the database?**
Browsers can't hold native DB connections or raw sockets. You'd need a DB credential in public code, and every user would open their own connection, overwhelming the DB. The backend holds credentials safely and shares a connection pool.

**Q: Where does frontend code actually execute?**
In the user's browser. The server only delivers HTML/CSS/JS. This is the opposite of a backend, where the server executes the logic and returns the result.

**Q: Why do browsers sandbox JavaScript?**
Because they run untrusted code from arbitrary servers. Isolation (no file system access, same-origin/CORS policy) stops a site from reading a user's files or data.
