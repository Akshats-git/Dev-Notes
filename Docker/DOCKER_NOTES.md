# Docker — Complete Notes (Part 1)

> Images, containers, port mapping, environment variables, writing a Dockerfile, and Docker Compose — explained with the *why*, not just the commands.

---

## Table of Contents

1. [The Problem Docker Solves](#1-the-problem-docker-solves)
2. [How Containers Fix It](#2-how-containers-fix-it)
3. [Installing Docker](#3-installing-docker)
4. [Images vs Containers](#4-images-vs-containers)
5. [Running Your First Container](#5-running-your-first-container)
6. [The Commands You Will Actually Use](#6-the-commands-you-will-actually-use)
7. [Docker Hub — The GitHub for Images](#7-docker-hub--the-github-for-images)
8. [Running Multiple Containers](#8-running-multiple-containers)
9. [Port Mapping](#9-port-mapping)
10. [Environment Variables](#10-environment-variables)
11. [Dockerizing a Node.js App](#11-dockerizing-a-nodejs-app)
12. [Layers and the Build Cache](#12-layers-and-the-build-cache)
13. [.dockerignore](#13-dockerignore)
14. [Docker Compose](#14-docker-compose)
15. [Quick Reference](#15-quick-reference)

---

## 1. The Problem Docker Solves

Picture a developer working on a project. On their machine everything runs perfectly. Their setup looks something like this:

```
  Developer A's laptop
  ────────────────────
  OS        : Windows 11
  Node.js   : v16
  MongoDB   : v5
  Redis     : v6

  → npm start → server is up and running 
```

Everything is fine. The trouble starts the moment a **second person** joins the team.

The new developer clones the repository and asks the obvious question: *what do I need to install?* And here is the first uncomfortable truth — **nobody actually remembers**. You installed MongoDB eight months ago and it has been quietly running in the background ever since. There are usually a handful of small services like that in any real project, and no single person has the complete list in their head.

Say you do manage to hand over the list. The second developer installs everything — but they are on macOS, and when you install software you naturally install the *latest* version:

```
  Developer B's laptop
  ────────────────────
  OS        : macOS
  Node.js   : v21   ← latest, not v16
  MongoDB   : v7    ← latest, not v5
  Redis     : v7    ← latest, not v6

  → npm start → 
```

The project almost certainly will not run on the first try. Not because the code is wrong, but because **the environment is different**. So they downgrade everything to match yours, which burns another afternoon — and if the project depends on anything platform-specific (a tool that only ships for Windows, say), no amount of downgrading will help. They simply cannot run that part locally.

Now scale this up:

- Deploying to the cloud? You have to reproduce the exact same setup on the server, by hand, and hope it starts.
- Using autoscaling? Every new machine that spins up needs the same manual ritual.
- Working in a team of fifty? Good luck communicating "install exactly these twelve things at exactly these versions" to fifty people, forever.

> **The core problem in one sentence:** *replicating an environment across machines is hard.* Everything above is a symptom of that one thing. Docker exists to solve it.

---

## 2. How Containers Fix It

Docker's answer is the **container**. A container is a small, isolated box that carries its own operating system, its own tools, and its own configuration — everything the application needs to run.

```
  ┌──────────────────────────────┐
  │   CONTAINER                  │
  │   ────────────────────────   │
  │   OS      : Ubuntu           │
  │   Node.js : v16              │
  │   MongoDB : v5               │
  │   Redis   : v6               │
  │   + your application code    │
  └──────────────────────────────┘
        │        │        │
        ▼        ▼        ▼
     Windows   macOS    Linux/Cloud
     (runs identically everywhere)
```

You describe the environment **once**. That description gets shared with your teammates and with your servers, and every one of them gets a byte-for-byte identical setup. Nobody installs Node or MongoDB on their own machine. Nobody argues about versions. The environment stops being something you *reproduce by hand* and becomes something you *ship*.

Containers are also deliberately lightweight — you can build one, destroy it, and rebuild it in seconds. That is what makes them practical to throw away and recreate constantly, which is exactly how they are used in real projects.

**Why this matters right now:** as the industry moves toward microservices and as more and more open-source projects ship with Docker configuration, "run it with Docker" has become the default onboarding instruction. Not knowing Docker turns into a small wall between you and a lot of codebases.

---

## 3. Installing Docker

You need two things, and the official installer ships both together:

| Thing | What it is |
|---|---|
| **Docker CLI** | The command-line tool. This is how you talk to Docker (`docker run`, `docker ps`, ...). |
| **Docker Desktop** | A GUI that shows your images and containers. Purely for visualisation and convenience. |

Download it from [docker.com](https://www.docker.com/) — pick your platform (Windows / macOS / Linux), install, and launch Docker Desktop once so the engine starts.

There is one more piece worth naming, because it explains what is actually happening under the hood:

```
  You type a command            The real work happens here
  ─────────────────             ──────────────────────────
                                
   docker CLI  ──────────────▶   Docker Daemon (dockerd)
                                 • pulls images
                                 • builds images
                                 • creates & destroys containers
                                 • starts & stops containers
                                       │
                                       ▼
                                 Docker Desktop (GUI)
                                 just *shows* you the state
```

The **daemon** is the actual Docker. The CLI is a messenger, and Docker Desktop is a window into what the daemon is doing. If Docker Desktop is not running, the daemon is not running, and your commands will fail with something like "cannot connect to the Docker daemon" — that is usually all that error means.

Verify the install:

```bash
docker            # prints the help text → CLI is installed
docker -v         # prints the version   → e.g. Docker version 24.x.x
```

---

## 4. Images vs Containers

This is *the* concept in Docker. Everything else is detail.

Think about your own laptop. It is a physical machine, and on top of it runs an operating system. The machine is the thing that executes; the OS is the thing being executed.

Docker splits the same idea in two:

- An **image** is the blueprint — the operating system plus the tools plus the configuration, frozen into a read-only package. An image does nothing on its own. It just sits there.
- A **container** is a running instance of an image — the actual machine doing the work.

```
              IMAGE  (ubuntu, node, mongo, redis, postgres...)
              the blueprint: OS + tools + config, read-only
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
  ┌────────┐   ┌────────┐   ┌────────┐
  │ CONT 1 │   │ CONT 2 │   │ CONT 3 │   ← three containers,
  │ data A │   │ data B │   │ data C │     one image
  └────────┘   └────────┘   └────────┘
   isolated     isolated     isolated
```

Two rules follow from this picture, and they explain most of Docker's behaviour:

**1. One image, many containers.** The same `ubuntu` image can back a hundred containers at once. That is the whole point — the blueprint is shared, the running copies are cheap.

**2. Containers are isolated from each other.** If you create a folder inside container 1, container 2 cannot see it. Anything you install, download, or break inside a container stays inside that container and never touches your host machine.

The everyday analogy: two laptops both running Windows. Same OS, same updates — completely separate files. Neither can read the other's data unless you deliberately connect them.

---

## 5. Running Your First Container

Let's run Ubuntu inside a container — on any host OS, including macOS or Windows.

```bash
docker run -it ubuntu
```

The first time you run this you will see an error-looking message that is not actually an error:

```
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
...
```

That is Docker telling you it doesn't have the Ubuntu image on your machine yet, so it is downloading one. Then your prompt changes to something like `root@a1b2c3d4e5f6:/#` — **you are now inside the container.**

Try a few things:

```bash
ls            # works — you're in an Ubuntu filesystem
whoami        # root
node          # "command not found" — Node isn't installed in this image
```

That last one is the isolation made visible. Node may well be installed on your actual laptop, but this container knows nothing about it. The container only has what its image gave it.

Press `Ctrl + D` (or type `exit`) to leave. The container stops.

### What that command actually did

```
docker run -it ubuntu
   │      │    │
   │      │    └── the IMAGE to use
   │      └─────── flags (explained below)
   └────────────── create a NEW container and start it
```

Step by step, Docker:

1. Checked whether the `ubuntu` image already exists on your machine.
2. It didn't, so it downloaded ("pulled") it from **Docker Hub**.
3. Created a brand-new container from that image.
4. Attached your terminal to it, because of `-it`.

Run the exact same command again and it is almost instant — the image is already local, so step 2 is skipped. And note that you now have **two** containers, not one. `docker run` always creates a *new* container.

### The `-it` flag

It is two flags written together:

| Flag | Stands for | What it does |
|---|---|---|
| `-i` | **i**nteractive | Keeps input open, so you can type into the container |
| `-t` | **t**ty | Gives you a terminal, so the output looks like a real shell |

Together they mean *"connect my terminal to the container's terminal and keep it connected."* Without them, the container starts and immediately exits because nothing is holding it open.

---

## 6. The Commands You Will Actually Use

Docker Desktop shows all of this visually, but on a cloud server there is no GUI — the CLI is all you have. These are worth memorising.

```bash
# List RUNNING containers
docker container ls
docker ps                       # the older, shorter alias — same thing

# List ALL containers, including stopped ones
docker container ls -a
docker ps -a                    # -a = all
```

Every container gets two identifiers: a long random **ID** (`a1b2c3d4e5f6...`) and a friendly random **name** like `pedantic_khorana` or `recursing_ramanujan`. You can use either one in any command, and you only need enough of the ID to be unique — the first few characters are plenty.

```bash
# Start a stopped container back up (keeps its data)
docker start <name-or-id>

# Stop a running container
docker stop <name-or-id>
```

> **`run` vs `start` — a distinction people get wrong constantly.**
> `docker run` creates a **brand-new** container from an image.
> `docker start` resumes an **existing** container you already created.
> If you keep using `run` you will quietly accumulate dozens of containers.

### Running a command inside a running container

```bash
# Run one command, print the result, come straight back
docker exec <name> ls

# Or open a shell INSIDE the container and stay there
docker exec -it <name> bash
```

The first form runs `ls` in the container and drops the output into *your* terminal — you never leave your own shell. The second attaches you to a Bash shell inside the container, and you stay there until you exit. This is how you inspect a container that is already running.

### Images

```bash
docker images                   # list images on your machine
docker image ls                 # identical
docker pull <image>             # download an image without running it
docker rmi <image>              # remove an image
docker rm <container>           # remove a container
```

---

## 7. Docker Hub — The GitHub for Images

[hub.docker.com](https://hub.docker.com) is to images what GitHub is to code: a public registry where images live and anyone can pull them. When you ran `docker run ubuntu`, this is where the image came from.

Browse it and you will find ready-made images for basically everything: `ubuntu`, `alpine`, `busybox`, `node`, `python`, `postgres`, `mongo`, `redis`, `nginx`.

This unlocks something genuinely useful. Suppose you need Node.js. You *could* start from `ubuntu` and install Node manually — or you could just use the image that already has it:

```bash
docker run -it node
```

The `node` image is heavier than `ubuntu` (it ships with Node pre-installed), but once it pulls you land directly in a Node REPL:

```js
> console.log("Hello from Docker")
Hello from Docker
> process.version
'v21.x.x'
```

Meanwhile, in a separate terminal on your actual machine:

```bash
node -v
# v18.x.x
```

Two completely different Node versions, running side by side, neither aware of the other. Extend the idea: need PostgreSQL for an afternoon? `docker run postgres`. You never install it on your laptop, and when you're done the container disappears without a trace.

### A word on trust

Docker Hub has two badges worth caring about: **Docker Official Image** and **Verified Publisher**. Anyone can push a public image, and a third-party image can contain anything — including malware. Prefer official and verified images. If you must use something else, look at who published it and what its Dockerfile does.

### Why companies publish images

When a company builds a self-hostable product, they typically ship an official image with the whole stack — app, database, cache, config — already wired together. Instead of a twenty-step setup guide, their docs just say:

```bash
docker run -p 3000:3000 <their-image>
```

That is the environment-replication problem being solved end to end. Self-hosting stops being a weekend project and becomes one command.

---

## 8. Running Multiple Containers

Open two terminals and run the same command in each:

```bash
docker run -it ubuntu
```

You now have two containers from one image. Prove they are isolated:

```bash
# In container 1
mkdir data1 && ls        # → data1

# In container 2
mkdir data2 && ls        # → data2   (no data1 anywhere)
```

Same image, same OS, completely separate filesystems. Confirm from your host:

```bash
docker container ls -a
```

```
CONTAINER ID   IMAGE     STATUS                     NAMES
a1b2c3d4e5f6   ubuntu    Up 2 minutes               pedantic_khorana
f6e5d4c3b2a1   ubuntu    Up 1 minute                recursing_ramanujan
9z8y7x6w5v4u   node      Exited (0) 5 minutes ago   nostalgic_hopper
```

Containers do use memory, so stop the ones you are not using — either from Docker Desktop, or:

```bash
docker stop $(docker ps -q)     # stop everything running
```

---

## 9. Port Mapping

Here is the problem that trips everyone up the first time.

You run a Node server inside a container on port 8000. It starts fine, the logs say `Server running on port 8000`. You open `http://localhost:8000` in your browser and get… nothing. Site can't be reached.

```
  Your machine                          Container
  ────────────                          ─────────
  browser → localhost:8000  ✗           Node server
                                        listening on :8000

  Isolated. The browser can't see in.
```

The server *is* running — but on port 8000 **inside the container**. Remember: containers are isolated. That isolation applies to the network too. Nothing on your host can reach into a container's ports unless you explicitly open a door.

That door is **port mapping**, and the flag is `-p`:

```bash
docker run -it -p 9000:9000 <image>
```

```
docker run -p 9000:9000
              │    │
              │    └── port INSIDE the container
              └─────── port on YOUR machine (the host)
```

Read it as: *"anything arriving at port 9000 on my machine, forward it to port 9000 inside the container."*

```
  Your machine                          Container
  ────────────                          ─────────
  browser → localhost:9000  ──┐
                              │  -p 9000:9000
                              └──────────────▶  :9000  Node server 
```

### The two ports don't have to match

This is the part worth understanding properly. The container's port is fixed by whatever the application inside was written to use — you cannot change it from outside. But the *host* port is entirely yours to choose:

```bash
docker run -it -p 6000:9000 <image>
```

Now the app still logs `Server running on port 9000` (it is, inside the container), but on your machine it answers at **`localhost:6000`**. `localhost:9000` gives you nothing, because you never opened that door.

This is what lets you run three copies of the same image at once — map them to 6000, 6001, 6002 on the host, all pointing at 9000 inside.

### Multiple ports

Repeat the flag as many times as you need. Some images serve more than one thing — an SMTP port and a web UI, for example:

```bash
docker run -p 1025:1025 -p 8025:8025 mailhog/mailhog
```

> **Rule of thumb:** if a container runs any kind of server and you want to reach it, you must map its port. No `-p`, no access. This is the single most common "why isn't my container working" cause.

---

## 10. Environment Variables

Applications need configuration that shouldn't be hardcoded — database URLs, API keys, feature flags, the port to listen on. You pass these into a container with `-e`, once per variable:

```bash
docker run -it \
  -e PORT=4000 \
  -e NODE_ENV=production \
  -e MONGO_URL=mongodb://localhost:27017 \
  <image>
```

```
-e KEY=VALUE
   │    │
   │    └── the value
   └─────── the variable name your app reads
```

Inside the container the app picks these up exactly as it always would:

```js
const port = process.env.PORT || 8000;
```

This is what makes one image usable in every environment. The image stays identical between your laptop, staging, and production — only the environment variables change. That is the whole idea behind configuration-through-environment, and Docker leans on it heavily.

---

## 11. Dockerizing a Node.js App

Time to build our own image. "Dockerizing" (or containerizing) an app means writing a recipe that turns your code into a shareable image.

### The app

```bash
mkdir docker-node && cd docker-node
npm init -y
npm install express
```

`main.js`:

```js
const express = require("express");
const app = express();
const PORT = process.env.PORT || 8000;

app.get("/", (req, res) => {
  return res.json({ message: "Hey, I am inside a container" });
});

app.listen(PORT, () => {
  console.log(`Server started on port ${PORT}`);
});
```

Nothing special — a plain Express server. Our folder now holds `main.js`, `package.json`, `package-lock.json`, and `node_modules/`.

### The Dockerfile

Create a file named exactly **`Dockerfile`** — capital D, **no extension**. Docker looks for this name by default.

This file is the recipe. It answers: what OS do we start from, what do we install, where does the code go, and what runs when the container starts.

```dockerfile
# 1. Base image — the foundation everything else sits on
FROM node:20

# 2. Where we'll work inside the container
WORKDIR /app

# 3. Copy dependency files FIRST (see the caching section below)
COPY package.json package-lock.json ./

# 4. Install dependencies inside the container
RUN npm install

# 5. Now copy the rest of the source code
COPY . .

# 6. Document which port the app uses
EXPOSE 8000

# 7. The command that runs when the container starts
CMD ["node", "main.js"]
```

### What each instruction means

| Instruction | What it does |
|---|---|
| `FROM` | The base image to build on top of. Every Dockerfile starts here. |
| `WORKDIR` | Sets the working directory inside the image; later commands run from here. Creates it if missing. |
| `COPY <src> <dest>` | Copies files from your machine into the image. |
| `RUN` | Executes a command **while building** the image (installing packages, etc.). |
| `EXPOSE` | Documents the port the app listens on. Informational — it does **not** publish the port; you still need `-p`. |
| `CMD` | The default command run when a **container starts**. Only the last `CMD` counts. |

> **`RUN` vs `CMD` — the difference that confuses everyone.**
> `RUN` happens at **build** time, and its result is baked into the image.
> `CMD` happens at **run** time, every time a container starts from that image.
> `npm install` is a `RUN`. `node main.js` is a `CMD`.

### Choosing a base image

We used `FROM node:20`, which comes with Node already installed. You *could* start lower down and do it by hand:

```dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get install -y curl
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
RUN apt-get install -y nodejs
```

It works, and it is a useful exercise for seeing that an image is just an OS you configure. But in practice there is no reason to — the official `node` image does the same thing, better maintained and better cached. Start as close to what you need as you can.

> **Tip:** `node:20-alpine` is a much smaller variant built on Alpine Linux. Same Node, a fraction of the size. Worth using once you're comfortable, keeping in mind Alpine ships a minimal userland so some native build tools need installing explicitly.

### Build and run

```bash
# Build an image from the Dockerfile in the current folder
docker build -t my-node-app .
```

```
docker build -t my-node-app .
                  │          │
                  │          └── build context: the folder to send to Docker
                  └───────────── -t = tag: the name you're giving the image
```

That trailing `.` is not decoration — it tells Docker which directory to use as the build context. Forgetting it is a very common error.

Confirm it exists, then run it:

```bash
docker images
docker run -p 8000:8000 my-node-app
```

Open `http://localhost:8000` and you get `{"message":"Hey, I am inside a container"}`. With an environment variable:

```bash
docker run -p 3000:4000 -e PORT=4000 my-node-app
```

The app listens on 4000 inside (because we told it to) and answers on 3000 on your machine.

That image is now the whole environment. Push it to Docker Hub and any teammate — on any OS — can run your app with one command, no Node installation required.

---

## 12. Layers and the Build Cache

Every instruction in a Dockerfile creates a **layer**, and Docker caches each one. On a rebuild, it reuses cached layers until it hits an instruction whose inputs changed — then that layer and **every layer after it** is rebuilt.

```
  FROM node:20                    ← layer 1   cached
  WORKDIR /app                    ← layer 2   cached
  COPY package*.json ./           ← layer 3   cached (package.json unchanged)
  RUN npm install                 ← layer 4   cached  ← the expensive one
  COPY . .                        ← layer 5   REBUILD (you edited main.js)
  CMD ["node", "main.js"]         ← layer 6   REBUILD
```

This is exactly why the Dockerfile above copies `package.json` on its own *before* copying the source code. Consider the alternative:

```dockerfile
COPY . .              # ← everything, including main.js
RUN npm install
```

Change one line of `main.js` and the `COPY . .` layer is invalidated — which means `npm install` runs again from scratch, every single build. On a real project that is minutes of waiting for no reason.

By copying only the dependency manifests first, `npm install` stays cached until your *dependencies* actually change. Editing source code then rebuilds in seconds.

> **The general rule:** order Dockerfile instructions from **least likely to change** to **most likely to change**. Base image and system packages at the top, your source code at the bottom.

---

## 13. .dockerignore

Right now `COPY . .` copies your entire folder into the image — including `node_modules/`, `.git/`, and any `.env` file sitting around. That is slow, bloats the image, and can leak secrets.

Create a `.dockerignore` next to your Dockerfile. It works just like `.gitignore`:

```
node_modules
npm-debug.log
.git
.gitignore
.env
Dockerfile
.dockerignore
README.md
```

Ignoring `node_modules` is not optional, by the way — it is correct. Dependencies get installed *inside* the container by `RUN npm install`, compiled for the container's OS. Copying your host's `node_modules` in would overwrite them with binaries built for the wrong platform, which is a genuinely confusing class of bug.

---

## 14. Docker Compose

Real applications are rarely one container. A typical setup is a Node API **and** MongoDB **and** Redis. You could start each by hand:

```bash
docker run -d -p 27017:27017 mongo
docker run -d -p 6379:6379 redis
docker run -d -p 8000:8000 -e MONGO_URL=... my-node-app
```

Three long commands, in the right order, remembered correctly, every time, by every person on the team. That does not scale, and it is exactly the kind of tribal knowledge Docker was supposed to eliminate.

**Docker Compose** lets you describe the whole setup in one YAML file and start it all with one command.

Create `docker-compose.yml`:

```yaml
services:
  app:
    build: .                    # build from the Dockerfile in this folder
    ports:
      - "8000:8000"
    environment:
      - PORT=8000
      - MONGO_URL=mongodb://mongo:27017/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      - mongo
      - cache

  mongo:
    image: mongo                # pull a ready-made image
    ports:
      - "27017:27017"

  cache:
    image: redis
    ports:
      - "6379:6379"
```

Then:

```bash
docker compose up          # build (if needed) and start everything
docker compose up -d       # same, but in the background (detached)
docker compose down        # stop and remove everything
docker compose logs -f     # follow the logs from all services
docker compose ps          # what's running
```

### The part that makes Compose click

Look at the connection string: `mongodb://mongo:27017/mydb`. Not `localhost` — **`mongo`**.

Compose puts every service on a shared network and makes each one reachable by its **service name**. The `app` container reaches the database at `mongo` and Redis at `cache`, because those are the keys under `services:`. Rename `cache` to `redis` and the hostname changes with it.

```
  ┌──────────── docker compose network ────────────┐
  │                                                 │
  │   app  ──── mongodb://mongo:27017 ────▶  mongo  │
  │     │                                           │
  │     └────── redis://cache:6379 ───────▶  cache  │
  │                                                 │
  └─────────────────────────────────────────────────┘
        │
        └── only "8000:8000" is exposed to your machine
```

`localhost` inside a container means *that container*, not your laptop and not a sibling container. Using `localhost` for a service running in another container is the single most common Compose mistake.

Note also that `ports` on `mongo` and `cache` is optional here. Containers on the same network already talk to each other freely — you only publish ports when *you* want to reach the service from your host (say, to connect MongoDB Compass to it).

`depends_on` controls **startup order** only. It waits for the container to *start*, not for the database to be *ready to accept connections*. Apps still need retry logic on their first connection; this trips up a lot of people.

This one file is the payoff for everything above. A new teammate clones the repo, runs `docker compose up`, and has the entire stack — right versions, right wiring — without installing Node, MongoDB, or Redis on their machine at all.

---

## 15. Quick Reference

### Mental model

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  DOCKER HUB  (hub.docker.com)                                    │
  │  public registry: ubuntu, node, mongo, redis, postgres, nginx... │
  └───────────────────────────┬──────────────────────────────────────┘
                              │  docker pull  /  automatic on run
                              ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  IMAGE — the blueprint (OS + tools + config + code), read-only   │
  │  built from a Dockerfile:  docker build -t myapp .               │
  └───────────────────────────┬──────────────────────────────────────┘
                              │  docker run
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ CONTAINER 1  │    │ CONTAINER 2  │    │ CONTAINER 3  │
  │ own data     │    │ own data     │    │ own data     │
  │ own network  │    │ own network  │    │ own network  │
  └──────────────┘    └──────────────┘    └──────────────┘
     isolated from each other and from your host

  To reach in:   -p host:container   (ports)
  To configure:  -e KEY=VALUE        (environment variables)
  Many at once:  docker compose up   (docker-compose.yml)
```

### Command cheat sheet

| Command | What it does |
|---|---|
| `docker -v` | Check the installed version |
| `docker run <image>` | Create and start a **new** container |
| `docker run -it <image>` | ...and attach your terminal to it |
| `docker run -p 8000:8000 <image>` | ...and map host port → container port |
| `docker run -e KEY=VAL <image>` | ...and pass an environment variable |
| `docker run -d <image>` | ...in the background (detached) |
| `docker ps` / `docker container ls` | List running containers |
| `docker ps -a` | List all containers, including stopped |
| `docker start <name>` | Restart an existing stopped container |
| `docker stop <name>` | Stop a running container |
| `docker rm <name>` | Delete a container |
| `docker exec <name> <cmd>` | Run one command inside a running container |
| `docker exec -it <name> bash` | Open a shell inside a running container |
| `docker images` | List local images |
| `docker pull <image>` | Download an image without running it |
| `docker rmi <image>` | Delete an image |
| `docker build -t <name> .` | Build an image from the Dockerfile here |
| `docker logs <name>` | See a container's output |
| `docker compose up` | Start everything in docker-compose.yml |
| `docker compose down` | Stop and remove it all |

### Dockerfile cheat sheet

| Instruction | When it runs | Purpose |
|---|---|---|
| `FROM` | build | Base image to start from |
| `WORKDIR` | build | Set the working directory inside the image |
| `COPY` | build | Copy files from host into the image |
| `RUN` | build | Execute a command and bake the result in |
| `ENV` | build | Set an environment variable in the image |
| `EXPOSE` | build (docs only) | Declare the port the app uses |
| `CMD` | **container start** | Default command to run |

### Common mistakes

| Symptom | Cause |
|---|---|
| Container exits immediately | No `-it`, and nothing keeping it alive |
| `localhost:PORT` doesn't respond | Forgot `-p`, or mapped the wrong host port |
| Containers pile up | Using `docker run` when you meant `docker start` |
| `npm install` reruns on every build | `COPY . .` placed before `RUN npm install` |
| App can't reach the database in Compose | Used `localhost` instead of the service name |
| Huge image / weird native module errors | No `.dockerignore`; host `node_modules` copied in |
| "Cannot connect to the Docker daemon" | Docker Desktop / the daemon isn't running |

---

## Coming in Part 2

- **Docker networking** — bridge, host, and custom networks in detail
- **Volume mounting** — persisting data beyond a container's lifetime, and live-reloading code during development
- **Multi-stage builds** — building in one image, shipping from a much smaller one

---

*Notes from "Docker In One Shot — Part 1".*
