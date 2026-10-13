# 28 — DevOps for Backend Engineers

**Video:** [#30 DevOps for backend engineers](https://www.youtube.com/watch?v=_E8X5RIJfQ4) (1h 48m)

The final lecture. Everything built so far runs on your laptop at `localhost`. How does it become reachable over the internet, stay alive while you ship changes daily, and survive failures? This is **DevOps**. The speaker is explicit: the goal is **a mental model**, not a deep dive on AWS or CI/CD tooling. After this you shouldn't be clueless when someone mentions any of it.

---

## Contents

1. [What DevOps is, and why](#what-devops-is-and-why)
2. [DORA metrics: speed and stability aren't a trade-off](#dora-metrics-speed-and-stability-arent-a-trade-off)
3. [Continuous integration and branching](#continuous-integration-and-branching)
4. [Builds and pipelines](#builds-and-pipelines)
5. [GitHub Actions and OIDC](#github-actions-and-oidc)
6. [Versioning a deployment](#versioning-a-deployment)
7. [Containers](#containers)
8. [Container images and registries](#container-images-and-registries)
9. [What runs a container: Docker, containerd, runc, OCI](#what-runs-a-container-docker-containerd-runc-oci)
10. [A good container image](#a-good-container-image)
11. [Running on one machine](#running-on-one-machine)
12. [Multiple machines: Kamal and immutable infrastructure](#multiple-machines-kamal-and-immutable-infrastructure)
13. [Kubernetes](#kubernetes)
14. [Delivery vs deployment, and rollout strategies](#delivery-vs-deployment-and-rollout-strategies)
15. [GitOps, environments and promotion](#gitops-environments-and-promotion)
16. [Infrastructure as code](#infrastructure-as-code)
17. [Operations: SLI, SLO, SLA, error budgets, on-call, incidents](#operations-sli-slo-sla-error-budgets-on-call-incidents)
18. [Chaos engineering and supply chain](#chaos-engineering-and-supply-chain)
19. [Summary](#summary)
20. [Interview questions](#interview-questions)

---

## What DevOps is, and why

[0:00](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=0s)

**DevOps = Development + Operations.** The outcome: *a commit becomes a running service on the internet, without human involvement and without downtime.*

### The old world

| Team | Measured by | Wants |
|---|---|---|
| **Developers** | How fast features reach production | **More** deployments |
| **Ops / infra** | How rarely production goes down | **Fewer** deployments (each deploy adds entropy) |

A conflict of interest. When something breaks, devs say "it's the infrastructure", ops say "it's your code". Releases slip to once a week or once in two weeks, and everyone must be present.

### The DevOps ideas

- Infrastructure as **code**, not hand-configured servers.
- Code **and** infra config in the same version control.
- **One command to build, one to deploy**.
- **Feature flags** to enable/disable features without deploying.
- **Shared observability** (logs, alerts, CPU/memory metrics) so developers see how their code behaves in production.

> What solved it was making deployment **small, cheap and boring**: one button / merging a branch, seconds to start. Then people deploy several times a day without hesitation.

---

## DORA metrics: speed and stability aren't a trade-off

[6:38](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=398s)

The DORA research team surveyed tens of thousands of teams. Four numbers:

| Type | Metric | Meaning |
|---|---|---|
| Speed | **Change lead time** | Commit → live in production (merged 9:00, live 11:00 = 2 h) |
| Speed | **Deployment frequency** | How often you deploy |
| Stability | **Change failure rate** | % of deploys that cause failures (4 of 100 = 4%) |
| Stability | **Failed-deployment recovery time** | How long to recover |

(2024) Top group (~20% of teams): deploy on demand, commit→prod in under a day, ~5% failure, recovery < 1 hour. Bottom group: deploy every 1–6 months, ~40% failure, a week to recover.

> The teams that deploy **more** are also **more stable**. Speed and stability are **not** a trade-off.

**Why:** a monthly deploy bundles ~200 commits from 10–20 people. When it breaks, finding the culprit takes a day. A deploy of one change by one person an hour ago makes the cause obvious, and the author still remembers it. **Deploy as often and as small as possible.**

---

## Continuous integration and branching

[11:15](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=675s)

**CI** is a **practice**: every developer merges into one shared branch **at least once a day**, and each merge is checked by an automated build running all the tests. (The "CI server" is the tool enforcing the checks, not the practice.)

Why daily: two people on 3-week feature branches face hours of merge conflicts. Merging daily means small drift, and conflicts take minutes.

### Branching model

- The popular **Git-flow** (develop, release, hotfix, main) was built for software shipped as versions customers install (desktop/mobile), where you support v2 while building v3. Its own author later wrote that for web apps that ship continuously you should use something simpler.
- A backend has **one version in production: the latest**. Use **trunk-based development**: one `main`; short-lived feature branches (a day or two) that are merged and deleted; release from `main`.
- Feature taking 2–3 weeks? Merge small pieces within a day or two and keep them behind a **feature flag**.

---

## Builds and pipelines

[15:54](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=954s)

A **build** turns source into something runnable: compile, fetch dependencies, package into an **artifact** (binary/executable). It should be **reproducible** (not dependent on what's installed on the machine; same source + config → byte-identical result), so pin the compiler/runtime version and use lock files.

- Go: `go build` with a pinned Go version and `go.sum` committed.
- Node: `npm ci` with the lock file.
- Stamp the **commit hash** into the artifact so a running server can report which commit it runs.

**Deployment pipeline** = build + test stages in sequence. Early stages are **fast and cheap** (compile + unit tests, minutes; fail → stop immediately and tell the dev). Later stages are slower and costlier (integration tests with real DBs, deploy to staging, verification), and the last stage is production.

---

## GitHub Actions and OIDC

[19:31](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=1171s)

| Term | Meaning |
|---|---|
| **Workflow** | A YAML file in `.github/workflows/` |
| **Event** | Trigger: push, pull request, schedule, manual click |
| **Job** | Steps that run on one machine. Jobs can run in parallel or `needs` each other |
| **Runner** | A **fresh VM** for the job, discarded afterwards (or self-hosted) |
| **Step** | A command or action |

Runners are fresh each time, so every workflow starts with a `checkout`. Demo: `test` job → `build` job (waits, builds the binary, uploads it as an artifact) → `image` job (waits for both).

### Cloud credentials: OIDC beats repo secrets

The pipeline runs on GitHub's infrastructure, but your cloud is elsewhere, so it needs credentials.

- **Repository secrets** (old way): every job can read them. If leaked, valid until rotated.
- **OIDC (OpenID Connect)**: when a job starts, GitHub's identity provider issues a **short-lived signed token** saying "this job, in this repo, on this branch, in this workflow". Your cloud (e.g. AWS) is configured to **trust GitHub's tokens only for that repo/branch**, and exchanges it for **credentials that expire when the job ends**. No permanent cloud credential is stored anywhere.

---

## Versioning a deployment

[24:07](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=1447s)

**Semantic versioning (major.minor.patch)** is for **libraries** whose users need to know about breaking changes. Your deployed service's users are HTTP clients who don't care about the deployment version (route versioning is a separate thing).

For deployments, stamp the **commit hash** (and optionally a build ID): in the artifact, as the Docker image tag, and exposed by a health/info endpoint, so after deploy you can verify the latest code is actually running.

---

## Containers

[26:38](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=1598s)

Problem: the binary works on my laptop but fails on the server (older system library, no timezone DB, Node 18 vs 22). Solution: **ship the environment with the code**: runtime, libraries, files, exact versions, in one unit. That unit is a **container**.

### Container ≠ small VM

| | Virtual machine | Container |
|---|---|---|
| Kernel | Has its **own** OS kernel | **No own kernel.** An ordinary process on the host's kernel (visible in `ps` on the host) |
| Isolation | Very strong | Process-level, via kernel features |
| Weight | GBs of RAM, slow boot, own disk | Light |

What makes a process a "container" is the **special treatment from the host kernel**: its own process IDs, hostname, filesystem root, network interfaces, and limits on CPU/memory. Three Linux kernel features:

### 1. Namespaces (what a process can *see*)

[31:13](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=1873s)

A namespace wraps a global resource so a group of processes sees its own copy:

| Namespace | Isolates |
|---|---|
| **PID** | process IDs (inside, your process is PID 1; the outside is invisible) |
| **UTS** | hostname |
| **Mount** | filesystem mount table |
| **Network** | interfaces, ports, routes (two containers can both listen on :80) |
| IPC, user, cgroup, time | (about eight in total) |

Three system calls manage them: **`clone`** (new process, with flags), **`unshare`** (move the calling process into a new namespace), **`setns`** (join an existing one). All container tooling (Docker, Kubernetes) uses them.

**Toy container (Liz Rice, *Containers from Scratch*, ~76 lines of Go):** the parent calls `clone` with `NEWUTS | NEWPID | NEWNS`, re-runs itself as a child. The child (PID 1) sets hostname `container`, `chroot`s into an Ubuntu rootfs, mounts a fresh `/proc`, and runs a shell. Inside: `hostname` → `container`, `ps` → just 2–3 processes, and on the host it's an ordinary process.

### 2. cgroups (what a process can *use*)

[36:21](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=2181s)

Namespaces don't limit resource use. A container could still eat all host RAM. **Control groups** do: `/sys/fs/cgroup` is a file system where creating a **folder** creates a group. Files inside: `memory.max`, `cpu.max`, `pids.max`. Write a limit → all processes in the group share it. Write a PID into `cgroup.procs` → it joins.

When Kubernetes says "memory limit 512 MiB", it writes that number to `memory.max`. Exceed it → kernel **OOM killer** kills the process → the famous `OOMKilled` status. (Demo: memory climbs to the limit then the process is killed; the kernel log says why.)

### 3. Layered file system (OverlayFS)

[38:25](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=2305s)

Copying a 100 MB Ubuntu root for every container would be wasteful. Container filesystems are **layers** (tarballs of changes: files added/modified/deleted). **OverlayFS** stacks **read-only lower layers** + **one writable upper layer** into a single view. Read → find the file in the topmost layer that has it. Write → copy-up to the upper layer and write there. Ten containers from one image share every lower layer and each gets a thin writable layer. Stop the container → the writable layer is gone, so **container filesystems are disposable** (persist elsewhere; see Factor 6 in [25](25-twelve-factor-app.md)).

---

## Container images and registries

[40:29](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=2429s)

A **blob** is unstructured binary data (a PDF, a layer). An **image** = **layers (blobs, identified by SHA-256)** + a **config** (JSON: ordered layer list, command, env, working dir, ports) + a **manifest** (JSON: points to config and layers with size and hash). The image identity is the **hash of the manifest**. Tags like `ubuntu:24.04` or `myapp:v1` just point to a digest. Think of an image as **a Git commit for your file system**.

### Dockerfile and caching

Each file-system-changing instruction creates a layer: `FROM` (base), `COPY`, `RUN`... The builder caches layers **from the top until the first instruction whose inputs changed**, and everything below is rebuilt. So **order matters**:

```dockerfile
# GOOD: dependencies change rarely, code changes constantly
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

Copy the whole source first and every code change invalidates the dependency install.

**Multi-stage builds** (compiled languages): stage 1 has the compiler and builds the binary; stage 2 starts from a tiny base image and copies only the binary → a much smaller image without build tools.

### Registry

A **registry** is an HTTP server of **blobs** (fetched by digest) and **manifests** (by tag or digest). Why: the build and run stages may run on different machines, so after building you **push** the image and before running you **pull** it. Push uploads only layers the registry doesn't already have. Pull downloads the manifest, then only the missing layers (so a small code change deploys in seconds). Examples: Docker Hub, GitHub's registry, AWS ECR, GCP, Azure. You can run your own.

---

## What runs a container: Docker, containerd, runc, OCI

[49:10](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=2950s)

`docker run` doesn't make the syscalls itself:

```
docker (CLI client) ─socket─► dockerd (daemon)
                                  └─► containerd (image & container lifecycle: pull layers, overlay mount, keep running)
                                         └─► runc (reads an OCI JSON spec; does clone/unshare + writes cgroup files + changes root + execs the process, then exits)
                                                 └─► a tiny "shim" stays as parent
```

The **Open Container Initiative (OCI)** standardises the image layout, how a runtime is told to run it, and how a registry serves it. So an image built with Docker runs in a Kubernetes cluster that has no Docker daemon (Kubernetes talks to containerd via **CRI**, and containerd to runc). The `runc` C code does exactly the namespace creation of the toy program.

---

## A good container image

[51:44](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=3104s)

| Property | Why |
|---|---|
| **Small** | Fewer files to update, less to download per deploy |
| **One process** | Just your service. No init system, no cron inside |
| **Non-root** | If your code is compromised it doesn't get root on the container |
| **Pinned** | Specific base version, not `latest`, so builds are reproducible |
| **Scanned** | Base-image packages accumulate known vulnerabilities weekly. Scanners match the package list to published CVEs |

**Buildpacks / Nixpacks:** PaaS platforms (Render, Railway...) detect your language from `package.json`, `go.mod`, `requirements.txt`, `Cargo.toml` and build an optimised image **without a Dockerfile**.

---

## Running on one machine

[54:17](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=3257s)

Three things:

1. **A process the OS keeps running** and restarts when it dies → **systemd** (the init system in every mainstream Linux). A *unit file* says what to run, as which user, restart on failure. Stdout goes to the **journal** (`journalctl -u <unit>`). The `ExecStart` is your `docker run ...`.
2. **A reverse proxy** on 80/443 terminating TLS and forwarding to the service port → **Nginx** or **Caddy**. Firewall allows 80/443; DNS points your domain at the machine.
3. **Logs** going somewhere readable.

### HTTPS certificates: ACME and Let's Encrypt

Getting a TLS certificate used to be hectic; now it's an automated protocol, **ACME**, run at scale by **Let's Encrypt** (free CA):

```
1. Server's ACME client generates a key; asks the CA for a certificate for domain X.
2. CA replies with a challenge: "serve this random token at http://X/.well-known/... on port 80" (or publish it as a DNS record).
3. Client does it. CA fetches the token from several vantage points.
4. CA is convinced you control X → signs → certificate returned.
5. Valid 90 days (short on purpose, so a stolen key is useful for a limited time). Auto-renewed ~every 60 days.
```

**Caddy** has ACME built in: give it the domain and it obtains and renews certificates.

---

## Multiple machines: Kamal and immutable infrastructure

[58:52](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=3532s)

Scaling horizontally (see [19](19-scaling-and-performance-part-1.md)) means deploying to 3 machines **with zero downtime**. **Kamal** automates it over SSH: install Docker if needed → log in to the registry → build and push the image → on each host pull the image and start the **new** container while the old one runs → wait for `/health` to return success → tell the proxy to route to the new container → stop the old one.

### Don't keep pet servers

[1:00:55](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=3655s)

A hand-maintained server that only 1–2 people know how to repair is the **snowflake** from the 12-factor video. Rules:

- Never maintain servers by hand; never SSH in to patch one.
- Change the **code/config** and redeploy, replacing instances.
- **Immutable infrastructure:** a server is never modified after it starts, only **replaced**. Containers make this cheap (you replace an image, not an OS install).

Everything that follows (clusters) depends on the assumption that servers are **cheap, replaced and not modified**.

---

## Kubernetes

[1:03:28](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=3808s)

With 40 containers on 10 machines: which runs where? Who restarts a dead one? Who adds capacity on spikes? Who replaces 40 old containers with 40 new ones in order, without downtime? An **orchestrator**: **Kubernetes** (the open-sourced version of Google's internal system).

> **Declarative:** you don't say what to do; you describe the **desired end state** ("3 copies of this image, port X, this much memory") and Kubernetes works out the steps.

### Architecture

| Component | Role |
|---|---|
| **API server** | The front end; the only thing that talks to the store |
| **etcd** | Distributed key-value store holding the desired state |
| **Controllers** | Loops that compare *what you asked for* with *what exists* and fix the gap (asked for 3, 2 are running → start one) |
| **Scheduler** | Picks a machine for each new pod |
| **Controller manager** | Runs the built-in controller loops |
| **kubelet** (on each node) | Asks the API server "what should run here?" and makes it so |
| **kube-proxy** | Networking for services |

### Objects

[1:04:59](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=3899s)

| Object | What it is |
|---|---|
| **Pod** | The unit that's scheduled: one (usually) or more containers sharing a network namespace and volumes. Has its own IP (so two pods can both listen on 8080). Ephemeral. Dies with its machine. Sidecars (log shipper, proxy) are why pods can hold several containers |
| **Labels** | Key-value tags (`app=api`, `version=v2`) used to select pods |
| **ReplicaSet** | Keeps *n* pods running from a template |
| **Deployment** | Owns ReplicaSets. Change the template (new image) → creates a new ReplicaSet and shifts the count. **You write Deployments, never ReplicaSets or bare pods** |
| **Service** | A **stable virtual IP + DNS name** for a set of pods (selector-based). Pods come and go; the Service address doesn't |
| **Ingress / Gateway API** | Maps a hostname + path from outside to a Service; a controller programs an actual load balancer |
| **ConfigMap** | Non-sensitive config as env vars/files |
| **Secret** | Sensitive values, given only to pods that ask |
| **resources: requests & limits** | `requests` = reserved on a machine; `limits` = hard cap (enforced by cgroups: exceed memory → OOMKilled → kubelet restarts the pod) |

Full chain: **domain → load balancer → Service → Pods.** Pods change every deploy; everything in front stays static.

### Rolling update

[1:10:37](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=4237s)

New image → Deployment makes a new ReplicaSet and shifts counts step by step, bounded by **`maxSurge`** (extra pods above the desired count) and **`maxUnavailable`** (pods below it), both **25%** by default. With 3 replicas: scale new +1, wait until ready → scale old −1 → repeat until new=3, old=0.

Safety: a new pod gets **no traffic until it reports ready**, so a broken image can only stall the rollout, not take the service down. The old ReplicaSet stays at 0 replicas (not deleted), making **rollback** trivial.

### Probes

[1:12:38](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=4358s)

The kubelet checks your container on a schedule (run a command, GET a path, or open a TCP port):

| Probe | Question | On failure |
|---|---|---|
| **Readiness** | Can this pod take traffic now? | Removed from Service endpoints. Re-added when it passes. **No restart** |
| **Liveness** | Is it alive or stuck? | Kubelet **kills and restarts** the container |
| **Startup** | Has a slow-starting process finished booting? | Holds the other two probes until it has |

> **Pitfall:** don't put a DB dependency in the liveness probe. A slow DB makes probes time out → all pods restart in a wave → more load → slower DB. Liveness checks "stuck?"; readiness checks "can I serve now?"

### Scaling

Scale = change a number (`replicas: 10`) and the controller starts 7 more. The **Horizontal Pod Autoscaler (HPA)** adjusts replicas automatically from average CPU (with min/max you set).

**Do you need Kubernetes?** Start small and *earn* the complexity: one machine with systemd or Docker Compose or a hosting platform → several machines with Kamal → Kubernetes when it's natural.

---

## Delivery vs deployment, and rollout strategies

[1:16:12](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=4572s)

| Term | Meaning |
|---|---|
| **Continuous delivery** | Software is **always in a deployable state**: any commit can go to production by pressing a button. A *technical* property |
| **Continuous deployment** | The button press itself is automated: every change that passes goes to production automatically, many times a day. A *product/business decision* (a team may choose weekly releases for changelogs/compliance) |

You can't have continuous deployment without continuous delivery.

### Strategies

| Strategy | How | Trade-off |
|---|---|---|
| **Recreate** | Stop all old, start all new | Simple, but a **downtime gap** |
| **Rolling update** (default in K8s) | Replace one by one, gated by readiness | Zero downtime, but **both versions serve traffic at once** for a while (requires compatibility) |
| **Blue-green** | Two full environments. Deploy to idle **green**, test, then switch the router. **Instant rollback** by switching back | **Double infrastructure cost** while both run. Only one version at a time |
| **Canary** | Deploy to 1 of 10 pods, send ~10% of traffic; compare errors/latency vs the old; widen to 20 → 50 → 75 → 100%, or reroute away on bad numbers | Needs good metrics |
| **Progressive delivery** | A controller **automates** the canary: widens on a schedule, compares metrics at each step, rolls back on failure | |

---

## GitOps, environments and promotion

[1:21:47](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=4907s)

**GitOps** is a workflow:

1. The **desired state** of the whole system (deployments, services, config maps) is in **declarative files in Git**: versioned, reviewable, reversible.
2. An **agent inside the cluster** (**Argo CD**, **Flux**) pulls the repo and keeps the cluster matching it. Anyone who changes the cluster by hand has it **auto-corrected back**. Git is the single source of truth.

Every deploy is then a **pull request that changes the image digest**; merging *is* the deploy. The CI pipeline **never holds cluster credentials**: it only pushes the image and opens the PR. A balanced middle ground between "systemd on a box" and a fully human-managed K8s cluster.

**Environments** (dev / staging / production) run the same code and image with different config. Two phrases:

- **Preview environment:** a temporary live environment created per pull request for reviewers, destroyed on merge/close.
- **Promotion:** moving the same artifact from one environment to the next, usually with a human approval gate.

---

## Infrastructure as code

[1:24:51](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=5091s)

Question: could you rebuild your entire infrastructure in a brand-new cloud account in 1 hour, identical to today's? If it was set up by hand via the console, almost certainly not. **Infrastructure as Code (IaC):** machines, networks, permissions and config are in version-controlled files; a tool applies them and gives the same result however many times it's run.

| Category | Job | Tool |
|---|---|---|
| **Provisioning** (the one you'll use most) | Create machines, networks, cloud services | **Terraform** / **OpenTofu** (the open-source fork after Terraform's licence changes) |
| **Configuration management** | Install/configure software on a machine from a script | **Ansible** |
| **Image baking** | Build a VM image with everything pre-installed | **Packer** (+ `cloud-init` for first-boot scripts) |

### Terraform: plan / apply and state

You write resource blocks → `terraform plan` shows what it **will** create/change/destroy → approve → `terraform apply`. It talks to AWS/GCP/Azure APIs for you.

**State:**

- Terraform records the cloud provider's ID for each resource in a **state file**, so it knows "this block = that machine". On each run it compares state ↔ actual cloud ↔ your code; the diff is the plan.
- **Lose the state** → Terraform thinks nothing exists → plans to create everything again.
- **Two people with separate state copies** overwrite each other → keep state in a **remote store with locking**, not on laptops.
- The state contains **every attribute including secrets** → never commit it to Git; store it encrypted.

### Drift

Someone changes infra directly in the console/CLI (e.g. opens port 22). Your code and state don't know. The next `plan` sees something not in the code and plans to **remove** it. In the demo: no changes → open port 22 by CLI → `plan` shows 1 change to close it → `apply` closes it. **Code (and state) is the source of truth.**

---

## Operations: SLI, SLO, SLA, error budgets, on-call, incidents

[1:32:56](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=5576s)

Everything so far is *internal*. Users only care about results, so we quantify them.

| Term | Meaning | Example |
|---|---|---|
| **SLI** (Service Level **Indicator**) | A measurement of something users care about | % of requests succeeding; % responding within 300 ms |
| **SLO** (Service Level **Objective**) | A **target** for an SLI **over a window** | 99% of requests succeed over 30 days |
| **SLA** (Service Level **Agreement**) | A **contract** with consequences if you miss it | refund / extended plan if availability < 99.9% |

(Note: the video's captions sometimes say "SLA" for SLI; the meaning is as above.)

### What nines mean

| Availability | Allowed downtime |
|---|---|
| 99.9% | ~43 min / month |
| 99.99% | ~52 min / year (~4.5 min / month) |

Each extra nine = **10×** more promise.

### Error budget

If the SLO is 99.9%, the allowed 0.1% failure is a **budget** you actively track and *spend* on deploys, maintenance, experiments, risky migrations. While there is budget left, ship fast and deploy many times a day. Once spent (say on the 23rd of the month), the team **freezes deployments** and puts all effort into reliability until it recovers.

### On-call and paging

When an SLI drops someone must respond: the **on-call** engineer (a weekly **rotation** in a team of three), with a **backup** engineer. **Paging** (from old pagers) is an urgent notification from an incident-management service on your phone/laptop, triggered by the observability alerts (see [16](16-logging-monitoring-observability.md)).

### Incident response

[1:39:07](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=5947s)

Roles:

- **Incident commander**: high-level view of symptoms and likely causes; doesn't fix.
- **Fixer(s)**: the engineers actually fixing it.
- **Communicator**: updates the public **status page** and internal managers, so the fixers aren't distracted.

Keep an **incident document** with the timeline. **First goal is not the root cause but to stop the damage**: roll back the last deploy, turn off a feature flag, disable the feature. Find the root cause after service is restored.

### Postmortem

After the outage a document records: the minute-by-minute **timeline**, symptoms, attempts and their effects, **user impact** (percentages), the **root cause**, and **next steps** so it can't recur. (Example: Cloudflare's November outage, where a database permission change caused a query to return duplicate rows, doubling a config file's size past a software size limit.)

**Blameless culture:** the postmortem blames systems, never people. If you blame "this person typed the wrong command", you never learn why they thought it was right, and every software failure is ultimately a human failure you can't prevent without understanding it.

### Operating principles

1. **Design for failure**: assume every machine and dependency will fail.
2. Keep deployment as simple as possible (files and config, no manual setup).
3. **Make one change at a time** so you know what broke it.

---

## Chaos engineering and supply chain

[1:44:13](https://www.youtube.com/watch?v=_E8X5RIJfQ4&t=6253s)

**Chaos engineering:** cause failure **on purpose**: kill a pod in production during business hours (watch the ReplicaSet replace it); take a machine out (pods reschedule); inject latency into a dependency (do timeouts trigger?). The point: find out whether your safety mechanisms *actually work* before a real outage. (Demo: delete 1 of 3 pods → a new one is pending then running.)

**Software supply chain:** most of your image isn't your code: the base image, the language runtime and hundreds of packages. Two failure types: a dependency vanishes/changes and breaks thousands of builds, or a dependency is **poisoned**. In **2024**, a compression library most Linux systems depend on shipped a **backdoor** inserted into the *release tarball*, not the source repo, so code reviewers never saw it.

**Defences:** **pin** every dependency and base image (never `latest`) and **scan** images for known vulnerabilities.

---

## Summary

1. **DevOps** closes the Dev-vs-Ops conflict by making deployment **small, cheap and boring**. **Speed and stability aren't a trade-off** (DORA: lead time, deploy frequency, change-failure rate, recovery time).
2. **CI**: merge daily; **trunk-based development**; feature flags for long work.
3. **Build** = reproducible artifact stamped with the commit hash. **Pipeline** = cheap/fast stages first. **GitHub Actions**: workflow/event/job/runner/step. Use **OIDC** over long-lived secrets.
4. **Containers** are processes isolated by Linux **namespaces** (view), **cgroups** (limits), **OverlayFS** layers (files). Not small VMs.
5. **Images** = layers + config + manifest, identified by hash. Order Dockerfile instructions for caching; use multi-stage builds; push to a **registry**. Docker → containerd → runc, standardised by **OCI**.
6. **One machine**: systemd + reverse proxy (Nginx/Caddy) + ACME/Let's Encrypt. **Few machines**: Kamal. **Immutable infrastructure**: replace, never patch.
7. **Kubernetes**: declare desired state; Pod → ReplicaSet → Deployment → Service → Ingress; requests/limits; **rolling update** (`maxSurge`/`maxUnavailable`), **readiness/liveness/startup probes**, HPA. Earn the complexity.
8. **Delivery** (always deployable) vs **deployment** (auto-deployed). **Rolling, blue-green, canary, progressive.** **GitOps** (Argo CD/Flux), preview environments, promotion.
9. **IaC**: Terraform/OpenTofu (plan/apply, **state**, drift), Ansible, Packer.
10. **Operations**: SLI/SLO/SLA, **error budgets**, on-call, incident roles, **blameless postmortems**.
11. **Chaos engineering** tests your safety nets. **Pin and scan** to defend the supply chain.

## Interview questions

**Q: What problem does DevOps solve?**
The conflict between developers (want frequent change) and operations (want stability), which slows releases. DevOps makes deployments small, automated and cheap so teams can have both speed and stability.

**Q: What are the DORA metrics?**
Change lead time, deployment frequency, change failure rate, and failed-deployment recovery time.

**Q: Why are frequent small deploys safer?**
Each deploy has few changes, so a failure's cause is obvious and fresh in the author's mind. Rollback is simple.

**Q: CI vs CD (delivery vs deployment)?**
CI: merge to a shared branch daily with automated tests. Continuous delivery: always deployable, with a human trigger. Continuous deployment: automatic release to production.

**Q: What is a container, technically? How is it different from a VM?**
An ordinary host process isolated by kernel features: namespaces (visibility), cgroups (resource limits) and an overlay file system. No separate kernel, so it's much lighter than a VM.

**Q: What do cgroups do?**
Limit and account for resources (CPU, memory, pids) for a group of processes. Exceed the memory limit → OOM kill.

**Q: Why order Dockerfile instructions carefully?**
Layer caching invalidates from the first changed instruction downward. Copy dependency manifests and install before copying source, to avoid reinstalling on every code change.

**Q: What is a multi-stage build?**
Build in a stage with the compiler, then copy only the artifact into a minimal final image, so the shipped image is small and contains no build tools.

**Q: What does a registry do and why is it needed?**
It stores image blobs and manifests so a build on one machine can be pulled by others. Pushing/pulling transfers only missing layers.

**Q: Pod vs Deployment vs Service?**
Pod: the schedulable unit of one or more containers. Deployment: manages ReplicaSets/rollouts of pods. Service: a stable address for a changing set of pods.

**Q: Readiness vs liveness probe?**
Readiness gates traffic. Failure removes the pod from the Service without restarting it. Liveness detects stuck containers. Failure restarts them.

**Q: Rolling vs blue-green vs canary?**
Rolling: replace gradually (mixed versions during rollout). Blue-green: a full parallel environment with an instant switch and rollback (2× cost). Canary: send a small share of traffic to the new version first and expand if metrics are healthy.

**Q: What is GitOps?**
Desired state lives declaratively in Git, and an in-cluster agent (Argo CD/Flux) continuously reconciles the cluster to it. Deploys are PR merges.

**Q: What is Terraform drift and how do you handle it?**
Infrastructure changed outside Terraform. The next `plan` shows the diff and would revert it. Make changes in code, and keep state remote, locked and secret-safe.

**Q: SLI vs SLO vs SLA, and what's an error budget?**
SLI: a measured indicator. SLO: a target for it over a window. SLA: a contractual promise with penalties. Error budget: the allowed failure (1 − SLO) that you spend on risk; when spent, you freeze deploys and focus on reliability.

**Q: What is the first goal in an incident?**
Stop the damage (rollback, disable a feature flag), then find the root cause.

**Q: Why blameless postmortems?**
To discover why a human action seemed right at the time, which is the only way to fix the system and prevent recurrence.

**Q: What is OIDC in CI and why is it better than stored cloud secrets?**
The CI provider issues a short-lived signed token identifying the job/repo/branch, and the cloud exchanges it for temporary credentials. No long-lived secret to leak.
