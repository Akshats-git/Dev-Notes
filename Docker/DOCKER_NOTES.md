# Docker Notes (Part 1)

> Images, containers, port mapping, environment variables, Dockerfiles and Docker Compose. Explained with the reasons behind each thing, not just the commands.

---

## Table of Contents

1. [The Problem Docker Solves](#1-the-problem-docker-solves)
2. [How Containers Fix It](#2-how-containers-fix-it)
3. [Installing Docker](#3-installing-docker)
4. [Images vs Containers](#4-images-vs-containers)
5. [Running Your First Container](#5-running-your-first-container)
6. [The Commands You Will Actually Use](#6-the-commands-you-will-actually-use)
7. [Docker Hub](#7-docker-hub)
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

Picture a developer working on a project. Everything runs perfectly on their machine. Their setup looks like this.

```
  Developer A's laptop
  ────────────────────
  OS        : Windows 11
  Node.js   : v16
  MongoDB   : v5
  Redis     : v6

  npm start  ->  server is up and running
```

Everything is fine so far. The trouble starts when a second person joins the team.

The new developer clones the repository. Then they ask the obvious question. What do I need to install?

Here is the uncomfortable truth. Nobody actually remembers. You installed MongoDB eight months ago. It has been quietly running in the background ever since. Real projects usually have a handful of small services like that. No single person has the full list in their head.

Say you do manage to hand over the list. The second developer installs everything. But they are on macOS. And when you install software you naturally install the latest version.

```
  Developer B's laptop
  ────────────────────
  OS        : macOS
  Node.js   : v21   <- latest, not v16
  MongoDB   : v7    <- latest, not v5
  Redis     : v7    <- latest, not v6

  npm start  ->  error
```

The project will almost certainly fail on the first try. The code is not wrong. The environment is different.

So they downgrade everything to match yours. That burns another afternoon. And if the project depends on something platform specific then downgrading will not help at all. A tool that only ships for Windows cannot be installed on a Mac. They simply cannot run that part locally.

Now scale this up.

- Deploying to the cloud? You have to build the same setup on the server by hand. Then you hope it starts.
- Using autoscaling? Every new machine needs the same manual ritual.
- Working in a team of fifty? You cannot tell fifty people to install twelve exact versions and keep doing it forever.

> **The core problem in one sentence.** Replicating an environment across machines is hard. Everything above is a symptom of that one thing. Docker exists to solve it.

---

## 2. How Containers Fix It

Docker's answer is the container. A container is a small isolated box. It carries its own operating system. It carries its own tools and its own configuration. It has everything the application needs to run.

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

You describe the environment once. That description is shared with your teammates and with your servers. Every one of them gets an identical setup.

Nobody installs Node or MongoDB on their own machine. Nobody argues about versions. The environment is no longer something you rebuild by hand. It becomes something you ship.

Containers are also very lightweight. You can build one and destroy it in seconds. That is why people throw them away and recreate them all the time.

**Why this matters right now.** The industry is moving toward microservices. More and more open source projects ship with Docker config. "Run it with Docker" is now the default onboarding instruction. Not knowing Docker becomes a small wall between you and a lot of codebases.

---

## 3. Installing Docker

You need two things. The official installer ships both together.

| Thing | What it is |
|---|---|
| **Docker CLI** | The command line tool. This is how you talk to Docker. |
| **Docker Desktop** | A GUI that shows your images and containers. It is only for viewing. |

Download it from [docker.com](https://www.docker.com/). Pick your platform. Install it. Then launch Docker Desktop once so the engine starts.

There is one more piece worth naming. It explains what is actually happening under the hood.

```
  You type a command            The real work happens here
  ─────────────────             ──────────────────────────

   docker CLI  ──────────────▶   Docker Daemon (dockerd)
                                 • pulls images
                                 • builds images
                                 • creates and destroys containers
                                 • starts and stops containers
                                       │
                                       ▼
                                 Docker Desktop (GUI)
                                 just shows you the state
```

The daemon is the actual Docker. It does all the work. The CLI is just a messenger. Docker Desktop is a window into what the daemon is doing.

If Docker Desktop is not running then the daemon is not running. Your commands will fail with "cannot connect to the Docker daemon". That error usually means nothing more than this.

Verify the install.

```bash
docker            # prints the help text, so the CLI is installed
docker -v         # prints the version, e.g. Docker version 24.x.x
```

---

## 4. Images vs Containers

This is the core concept in Docker. Everything else is detail.

Think about your own laptop. It is a physical machine. An operating system runs on top of it. The machine is the thing that executes. The OS is the thing being executed.

Docker splits the same idea in two.

- An **image** is the blueprint. It is the operating system plus the tools plus the configuration. It is frozen into a read only package. An image does nothing on its own. It just sits there.
- A **container** is a running instance of an image. It is the actual machine doing the work.

```
              IMAGE  (ubuntu, node, mongo, redis, postgres...)
              the blueprint: OS + tools + config, read-only
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
  ┌────────┐   ┌────────┐   ┌────────┐
  │ CONT 1 │   │ CONT 2 │   │ CONT 3 │   <- three containers
  │ data A │   │ data B │   │ data C │      from one image
  └────────┘   └────────┘   └────────┘
   isolated     isolated     isolated
```

Two rules follow from this picture. They explain most of Docker's behaviour.

**1. One image can back many containers.** The same `ubuntu` image can run a hundred containers at once. That is the whole point. The blueprint is shared and the running copies are cheap.

**2. Containers are isolated from each other.** Create a folder inside container 1 and container 2 cannot see it. Anything you install or download or break inside a container stays inside that container. It never touches your host machine.

Here is the everyday analogy. Two laptops both run Windows. Same OS and same updates. But the files are completely separate. Neither can read the other's data unless you deliberately connect them.

---

## 5. Running Your First Container

Let us run Ubuntu inside a container. This works on any host OS including macOS and Windows.

```bash
docker run -it ubuntu
```

The first run shows a message that looks like an error. It is not one.

```
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
...
```

Docker is telling you it does not have the Ubuntu image yet. So it downloads one.

Then your prompt changes to something like `root@a1b2c3d4e5f6:/#`. You are now inside the container.

Try a few things.

```bash
ls            # works, you are in an Ubuntu filesystem
whoami        # root
node          # "command not found", Node is not in this image
```

That last one shows the isolation clearly. Node may well be installed on your laptop. But this container knows nothing about it. A container only has what its image gave it.

Press `Ctrl + D` or type `exit` to leave. The container stops.

### What that command actually did

```
docker run -it ubuntu
   │      │    │
   │      │    └── the IMAGE to use
   │      └─────── flags (explained below)
   └────────────── create a NEW container and start it
```

Docker did four things.

1. It checked whether the `ubuntu` image already exists on your machine.
2. It did not exist. So Docker downloaded it from Docker Hub.
3. It created a brand new container from that image.
4. It attached your terminal to that container because of `-it`.

Run the same command again and it is almost instant. The image is already local so step 2 is skipped.

Notice that you now have two containers and not one. `docker run` always creates a new container.

### The -it flag

It is two flags written together.

| Flag | Stands for | What it does |
|---|---|---|
| `-i` | **i**nteractive | Keeps input open so you can type into the container |
| `-t` | **t**ty | Gives you a terminal so the output looks like a real shell |

Together they mean one thing. Connect my terminal to the container's terminal and keep it connected.

Without them the container starts and exits right away. Nothing is holding it open.

---

## 6. The Commands You Will Actually Use

Docker Desktop shows all of this visually. But a cloud server has no GUI. The CLI is all you have there. These are worth memorising.

```bash
# List RUNNING containers
docker container ls
docker ps                       # older shorter alias for the same thing

# List ALL containers including stopped ones
docker container ls -a
docker ps -a                    # -a means all
```

Every container gets two identifiers. One is a long random **ID** like `a1b2c3d4e5f6`. The other is a friendly random **name** like `pedantic_khorana` or `recursing_ramanujan`.

You can use either one in any command. You only need enough of the ID to be unique. The first few characters are plenty.

```bash
# Start a stopped container back up. It keeps its data.
docker start <name-or-id>

# Stop a running container
docker stop <name-or-id>
```

> **run vs start.** People get this wrong constantly. `docker run` creates a brand new container from an image. `docker start` resumes a container you already created. Keep using `run` and you will quietly pile up dozens of containers.

### Running a command inside a running container

```bash
# Run one command, print the result, come straight back
docker exec <name> ls

# Or open a shell INSIDE the container and stay there
docker exec -it <name> bash
```

The first form runs `ls` in the container. The output lands in your own terminal. You never leave your shell.

The second form attaches you to a Bash shell inside the container. You stay there until you exit. This is how you inspect a container that is already running.

### Images

```bash
docker images                   # list images on your machine
docker image ls                 # identical
docker pull <image>             # download an image without running it
docker rmi <image>              # remove an image
docker rm <container>           # remove a container
```

---

## 7. Docker Hub

[hub.docker.com](https://hub.docker.com) is to images what GitHub is to code. It is a public registry where images live. Anyone can pull from it. Your `ubuntu` image came from here.

Browse it and you will find ready made images for almost everything. There is `ubuntu` and `alpine` and `busybox`. There is `node` and `python` and `postgres` and `mongo` and `redis` and `nginx`.

This unlocks something genuinely useful. Suppose you need Node.js. You could start from `ubuntu` and install Node by hand. Or you could just use the image that already has it.

```bash
docker run -it node
```

The `node` image is heavier than `ubuntu` because Node is preinstalled. Once it pulls you land directly in a Node REPL.

```js
> console.log("Hello from Docker")
Hello from Docker
> process.version
'v21.x.x'
```

Now open a separate terminal on your actual machine.

```bash
node -v
# v18.x.x
```

Two different Node versions run side by side. Neither is aware of the other.

Extend the idea. Need PostgreSQL for an afternoon? Just run `docker run postgres`. You never install it on your laptop. When you are done the container disappears without a trace.

### A word on trust

Docker Hub has two badges worth caring about. One is **Docker Official Image**. The other is **Verified Publisher**.

Anyone can push a public image. A third party image can contain anything including malware. So prefer official and verified images. If you must use something else then check who published it and read its Dockerfile.

### Why companies publish images

Many companies build products you can self host. They usually ship an official image with the whole stack already wired together. That means the app and the database and the cache and the config.

So instead of a twenty step setup guide their docs just say this.

```bash
docker run -p 3000:3000 <their-image>
```

That is the environment problem solved end to end. Self hosting stops being a weekend project. It becomes one command.

---

## 8. Running Multiple Containers

Open two terminals. Run the same command in each one.

```bash
docker run -it ubuntu
```

You now have two containers from one image. Let us prove they are isolated.

```bash
# In container 1
mkdir data1 && ls        # -> data1

# In container 2
mkdir data2 && ls        # -> data2   (no data1 anywhere)
```

Same image and same OS. But the filesystems are completely separate.

Confirm this from your host.

```bash
docker container ls -a
```

```
CONTAINER ID   IMAGE     STATUS                     NAMES
a1b2c3d4e5f6   ubuntu    Up 2 minutes               pedantic_khorana
f6e5d4c3b2a1   ubuntu    Up 1 minute                recursing_ramanujan
9z8y7x6w5v4u   node      Exited (0) 5 minutes ago   nostalgic_hopper
```

Containers do use memory. So stop the ones you are not using. Do it from Docker Desktop or from the CLI.

```bash
docker stop $(docker ps -q)     # stop everything running
```

---

## 9. Port Mapping

Here is the problem that trips up everyone the first time.

You run a Node server inside a container on port 8000. It starts fine. The logs say `Server running on port 8000`. You open `http://localhost:8000` in your browser and get nothing. The site cannot be reached.

```
  Your machine                          Container
  ────────────                          ─────────
  browser -> localhost:8000  X          Node server
                                        listening on :8000

  Isolated. The browser cannot see in.
```

The server is running. But it is on port 8000 inside the container.

Remember that containers are isolated. That isolation covers the network too. Nothing on your host can reach a container's ports until you open a door.

That door is called **port mapping**. The flag is `-p`.

```bash
docker run -it -p 9000:9000 <image>
```

```
docker run -p 9000:9000
              │    │
              │    └── port INSIDE the container
              └─────── port on YOUR machine (the host)
```

Read it like this. Anything arriving at port 9000 on my machine should be forwarded to port 9000 inside the container.

```
  Your machine                          Container
  ────────────                          ─────────
  browser -> localhost:9000  ──┐
                               │  -p 9000:9000
                               └─────────────▶  :9000  Node server
```

### The two ports do not have to match

This part is worth understanding properly.

The container's port is fixed. It is whatever the application inside was written to use. You cannot change it from outside.

But the host port is entirely your choice.

```bash
docker run -it -p 6000:9000 <image>
```

The app still logs `Server running on port 9000`. That is true inside the container. But on your machine it now answers at `localhost:6000`.

Visiting `localhost:9000` gives you nothing. You never opened that door.

This is what lets you run three copies of the same image at once. Map them to 6000 and 6001 and 6002 on the host. All three point at 9000 inside.

### Multiple ports

Repeat the flag as many times as you need. Some images serve more than one thing. An SMTP port and a web UI is a common example.

```bash
docker run -p 1025:1025 -p 8025:8025 mailhog/mailhog
```

> **Rule of thumb.** If a container runs a server and you want to reach it then you must map its port. No `-p` means no access. This is the most common cause of "why is my container not working".

---

## 10. Environment Variables

Applications need configuration that should not be hardcoded. Things like database URLs and API keys and feature flags and the port to listen on.

You pass these into a container with `-e`. Use it once per variable.

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

Inside the container the app reads them the way it always would.

```js
const port = process.env.PORT || 8000;
```

This is what makes one image usable in every environment. The image stays identical on your laptop and on staging and in production. Only the environment variables change.

That idea is called configuration through environment. Docker leans on it heavily.

---

## 11. Dockerizing a Node.js App

Time to build our own image. Dockerizing an app means writing a recipe. The recipe turns your code into a shareable image.

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

Nothing special here. It is a plain Express server.

Our folder now holds `main.js` and `package.json` and `package-lock.json` and `node_modules/`.

### The Dockerfile

Create a file named exactly **`Dockerfile`**. Capital D and no extension. Docker looks for this name by default.

This file is the recipe. It answers four questions. What OS do we start from? What do we install? Where does the code go? What runs when the container starts?

```dockerfile
# 1. Base image. The foundation everything else sits on.
FROM node:20

# 2. Where we will work inside the container
WORKDIR /app

# 3. Copy dependency files FIRST. See the caching section below.
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
| `WORKDIR` | Sets the working directory inside the image. Later commands run from here. It creates the folder if it is missing. |
| `COPY <src> <dest>` | Copies files from your machine into the image. |
| `RUN` | Runs a command while building the image. Used for installing packages. |
| `EXPOSE` | Documents the port the app listens on. It does not publish the port. You still need `-p`. |
| `CMD` | The default command run when a container starts. Only the last `CMD` counts. |

> **RUN vs CMD.** This difference confuses everyone. `RUN` happens at build time and its result is baked into the image. `CMD` happens at run time and fires every time a container starts. So `npm install` is a `RUN`. And `node main.js` is a `CMD`.

### Choosing a base image

We used `FROM node:20`. That image already has Node installed.

You could start lower down and do it by hand instead.

```dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get install -y curl
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
RUN apt-get install -y nodejs
```

This works. It is a useful exercise. It shows you that an image is just an OS you configure.

But there is no real reason to do it. The official `node` image does the same job. It is better maintained and better cached. Start as close to what you need as you can.

> **Tip.** `node:20-alpine` is a much smaller variant built on Alpine Linux. Same Node at a fraction of the size. Use it once you are comfortable. Just remember that Alpine ships a minimal userland. Some native build tools need installing explicitly.

### Build and run

```bash
# Build an image from the Dockerfile in the current folder
docker build -t my-node-app .
```

```
docker build -t my-node-app .
                  │          │
                  │          └── build context: the folder to send to Docker
                  └───────────── -t means tag, the name for your image
```

That trailing `.` is not decoration. It tells Docker which directory to use as the build context. Forgetting it is a very common error.

Confirm the image exists and then run it.

```bash
docker images
docker run -p 8000:8000 my-node-app
```

Open `http://localhost:8000`. You get `{"message":"Hey, I am inside a container"}`.

Now try it with an environment variable.

```bash
docker run -p 3000:4000 -e PORT=4000 my-node-app
```

The app listens on 4000 inside because we told it to. It answers on 3000 on your machine.

That image is now the whole environment. Push it to Docker Hub and any teammate can run your app with one command. Their OS does not matter. They do not need Node installed.

---

## 12. Layers and the Build Cache

Every instruction in a Dockerfile creates a **layer**. Docker caches each one.

On a rebuild Docker reuses cached layers. It stops reusing when it hits an instruction whose inputs changed. That layer is rebuilt and so is every layer after it.

```
  FROM node:20                    <- layer 1   cached
  WORKDIR /app                    <- layer 2   cached
  COPY package*.json ./           <- layer 3   cached (package.json unchanged)
  RUN npm install                 <- layer 4   cached  <- the expensive one
  COPY . .                        <- layer 5   REBUILD (you edited main.js)
  CMD ["node", "main.js"]         <- layer 6   REBUILD
```

This is exactly why the Dockerfile above copies `package.json` on its own. It does that before copying the source code.

Consider the alternative.

```dockerfile
COPY . .              # everything, including main.js
RUN npm install
```

Change one line of `main.js` and the `COPY . .` layer is invalidated. So `npm install` runs again from scratch on every single build. On a real project that is minutes of waiting for no reason.

Copying only the dependency files first keeps `npm install` cached. It only reruns when your dependencies actually change. Editing source code then rebuilds in seconds.

> **The general rule.** Order your Dockerfile from least likely to change to most likely to change. Base image and system packages go at the top. Your source code goes at the bottom.

---

## 13. .dockerignore

Right now `COPY . .` copies your entire folder into the image. That includes `node_modules/` and `.git/` and any `.env` file lying around.

That is slow. It bloats the image. It can also leak secrets.

Create a `.dockerignore` next to your Dockerfile. It works just like `.gitignore`.

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

Ignoring `node_modules` is not optional. It is correct.

Dependencies get installed inside the container by `RUN npm install`. They are compiled for the container's OS. Copying your host `node_modules` in would overwrite them with binaries built for the wrong platform. That produces a genuinely confusing class of bug.

---

## 14. Docker Compose

Real applications are rarely one container. A typical setup is a Node API and MongoDB and Redis.

You could start each one by hand.

```bash
docker run -d -p 27017:27017 mongo
docker run -d -p 6379:6379 redis
docker run -d -p 8000:8000 -e MONGO_URL=... my-node-app
```

That is three long commands. They must be run in the right order. Every person on the team must remember them correctly every time. This does not scale. It is exactly the tribal knowledge Docker was meant to remove.

**Docker Compose** fixes it. You describe the whole setup in one YAML file. Then you start it all with one command.

Create `docker-compose.yml`.

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

Then use these commands.

```bash
docker compose up          # build if needed and start everything
docker compose up -d       # same but in the background
docker compose down        # stop and remove everything
docker compose logs -f     # follow the logs from all services
docker compose ps          # see what is running
```

### The part that makes Compose click

Look at the connection string. It says `mongodb://mongo:27017/mydb`. It does not say `localhost`. It says `mongo`.

Compose puts every service on a shared network. Each service is reachable by its **service name**.

So the `app` container reaches the database at `mongo`. It reaches Redis at `cache`. Those are the keys under `services:`. Rename `cache` to `redis` and the hostname changes with it.

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

Inside a container `localhost` means that container. It does not mean your laptop. It does not mean a sibling container. Using `localhost` for a service in another container is the most common Compose mistake.

Note that `ports` on `mongo` and `cache` is optional. Containers on the same network already talk to each other freely. You only publish a port when you want to reach the service from your host. Connecting MongoDB Compass to it would be one reason.

Also note what `depends_on` really does. It controls startup order only. It waits for the container to start. It does not wait for the database to be ready for connections. Your app still needs retry logic on its first connection. This trips up a lot of people.

This one file is the payoff for everything above. A new teammate clones the repo and runs `docker compose up`. They get the entire stack at the right versions with the right wiring. They never install Node or MongoDB or Redis at all.

---

## 15. Quick Reference

### Mental model

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  DOCKER HUB  (hub.docker.com)                                    │
  │  public registry: ubuntu, node, mongo, redis, postgres, nginx... │
  └───────────────────────────┬──────────────────────────────────────┘
                              │  docker pull, or automatic on run
                              ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  IMAGE. The blueprint: OS + tools + config + code. Read-only.    │
  │  Built from a Dockerfile:  docker build -t myapp .               │
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
| `docker run <image>` | Create and start a new container |
| `docker run -it <image>` | Also attach your terminal to it |
| `docker run -p 8000:8000 <image>` | Also map a host port to a container port |
| `docker run -e KEY=VAL <image>` | Also pass an environment variable |
| `docker run -d <image>` | Run it in the background |
| `docker ps` or `docker container ls` | List running containers |
| `docker ps -a` | List all containers including stopped ones |
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
| `RUN` | build | Run a command and bake the result in |
| `ENV` | build | Set an environment variable in the image |
| `EXPOSE` | build (docs only) | Declare the port the app uses |
| `CMD` | container start | Default command to run |

### Common mistakes

| Symptom | Cause |
|---|---|
| Container exits immediately | No `-it` and nothing keeping it alive |
| `localhost:PORT` does not respond | Forgot `-p` or mapped the wrong host port |
| Containers keep piling up | Used `docker run` when you meant `docker start` |
| `npm install` reruns on every build | `COPY . .` placed before `RUN npm install` |
| App cannot reach the database in Compose | Used `localhost` instead of the service name |
| Huge image or odd native module errors | No `.dockerignore` so host `node_modules` got copied in |
| "Cannot connect to the Docker daemon" | Docker Desktop is not running |

---

## Coming in Part 2

- **Docker networking.** Bridge and host and custom networks in detail.
- **Volume mounting.** Keeping data alive beyond a container. Also live reloading code during development.
- **Multi stage builds.** Build in one image and ship from a much smaller one.

---

*Notes from "Docker In One Shot, Part 1".*
