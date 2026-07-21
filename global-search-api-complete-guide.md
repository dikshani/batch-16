# Global Search API — Complete DevOps Journey
### From Local Testing to Fully Automated CI/CD Deployment on Kubernetes

This document is a complete, hands-on reference covering everything done to take the `global-search-api` service from running on a local machine, all the way to a **fully automated CI/CD pipeline** that builds, pushes, and deploys to Kubernetes with a single click.

Every command is explained — what it does, why it's needed, and what each flag/option means — so this document can be used as a learning reference, not just a copy-paste script.

---

## Table of Contents

**Overview**
- [1. Project Overview](#1-project-overview)

**Setup & Deployment Walkthrough**
- [Part A — Running the App Locally](#part-a-running-the-app-locally)
- [Part B — Containerizing with Docker](#part-b-containerizing-with-docker)
- [Part C — AWS Setup: IAM & ECR](#part-c-aws-setup--iam--ecr)
- [Part D — Building and Testing on an EC2 Server](#part-d-building-and-testing-on-an-ec2-server)
- [Part E — The Multi-Architecture Problem and Its Fix](#part-e-the-multi-architecture-problem-and-its-fix)
- [Part F — Manually Deploying to Kubernetes (EKS)](#part-f-manually-deploying-to-kubernetes-eks)
- [Part G — Automating Everything with Jenkins](#part-g-automating-everything-with-jenkins)

**Reference**
- [Common Errors and Fixes (Cheat Sheet)](#common-errors-and-fixes-cheat-sheet)
- [Full Commands Reference](#full-commands-reference)
- [Pending / Next Steps](#pending--next-steps)

---

## 1. Project Overview

**What the app is:** `global-search-api` — a Node.js/Express backend that searches stocks, F&O, indices, IPOs, mutual funds, and commodities. It lives inside the `project-cash` repository, under `src/api/search/`.

**Tech stack:**
- Node.js + Express
- Redis (multiple instances for different data types — stocks, F&O, etc.)
- PostgreSQL (present in config, though `app.js` doesn't connect to it directly)
- Configuration lives in `config.js`, split by environment: `dev`, `qa`, `prod`, `dr`

**Infrastructure:**
- **Docker image registry:** AWS ECR — repository name `cash-global-search-apis`
- **Deployment target:** AWS EKS cluster — `ventura-mvp-uat-cluster`, namespace `uatenv`
- **CI/CD:** Jenkins, running on the `centralised-monitoring-uat` EC2 server (Jenkins agent label: `monitoring-server`)

**Key architectural fact that caused a major issue:** The EKS cluster's worker nodes run on **ARM64 (AWS Graviton)** processors, while typical EC2 build machines run **x86_64/amd64**. This mismatch is covered in detail in Part E.

---

## Part A: Running the App Locally

### Why this step matters
Before introducing the complexity of Docker or Kubernetes, it's important to confirm the code runs correctly on its own — that dependencies install cleanly and the core logic works.

### Step 1 — Navigate to the project folder

```bash
cd ~/project-cash/src/api/search
```
**Explanation:** `cd` (change directory) moves your terminal's "current location" into the folder where the app's source code lives. All the following commands assume you're standing inside this folder.

### Step 2 — Install dependencies

```bash
npm install
```
**Explanation:** `npm` (Node Package Manager) reads the `package.json` file, which lists every library the app needs (Express, Redis client, Pino logger, etc.), and downloads them all into a folder called `node_modules`. Without this, the app can't even start — it would throw `Cannot find module` errors.

### Step 3 — Check/set the `.env` file

```bash
cat .env
```
**Explanation:** `cat` (concatenate) simply prints a file's contents to the screen. This lets you see what's currently inside `.env` without opening an editor.

The file should contain a line like:
```
ENV=qa
```
This tells the app which block of `config.js` to use (`dev`, `qa`, `prod`, or `dr`) — each has different database hosts, Redis endpoints, and ports.

> ⚠️ **Formatting matters for Docker later:** Node's `dotenv` package tolerates `ENV = qa` (with spaces), but Docker's `--env-file` flag does not — it requires exactly `ENV=qa` with no spaces. It's worth keeping this format from the start.

### Step 4 — Run the app

```bash
npm run dev
```
**Explanation:** This reads the `"dev"` script defined in `package.json` (which internally runs `nodemon app.js`). `nodemon` is a tool that watches your files and automatically restarts the app whenever code changes — useful during development.

On success, you'll see:
```
Server has been started at <PORT>
```

### Step 5 — Test the running app (in a separate terminal)

```bash
curl "http://localhost:<PORT>/globalsearch/v1/get?searchkey=reliance&page=0&size=15&type=1"
```
**Explanation:** `curl` sends an HTTP request from the terminal, just like a browser would. Here it's hitting the app's search endpoint with query parameters, to confirm the server responds correctly.

**Query parameters, explained (from `validation.js`):**

| Param | Meaning |
|---|---|
| `searchkey` | The text being searched (e.g. "reliance") |
| `page` | Page number of results, starting at 0 |
| `size` | How many results to return per page |
| `type` | Which category: 0=all, 1=stock, 2=fno, 3=indices, 4=ipo, 5=mf, 6=commodity |

### Common Issue: App crashes with a Redis timeout

If `.env`/config points to placeholder Redis IPs (`10.x.x.x`) that aren't reachable, the app can crash with:
```
ConnectionTimeoutError: Connection timeout
```
This happens because the Redis client throws an **unhandled async error** — it's not caught by the surrounding `try/catch` blocks in `app.js`, since the error arrives after the initial connection attempt, via an event.

**Temporary local-testing workaround** (must be reverted before deploying anywhere real): comment out the five Redis-connection `try/catch` blocks in `app.js` using `/* ... */`. This makes Redis-dependent response fields come back empty (`[]`) instead of crashing the whole app.

```bash
git checkout -- app.js
```
**Explanation:** `git checkout -- <file>` discards any uncommitted local changes to that specific file, restoring it to the last committed version. This is how the temporary Redis-disabling edit gets safely undone.

---

## Part B: Containerizing with Docker

### Why Docker
Docker packages the app — Node.js runtime, code, and all dependencies — into a single, portable **image**. That image behaves identically no matter which machine runs it (in theory — see Part E for the one big exception we hit).

### The Dockerfile (final, working version)

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app/api/search
COPY package*.json ./
RUN npm install --omit=dev

FROM node:18-alpine AS runner
RUN addgroup -g 1001 -S nodejs && adduser -S nodeuser -u 1001
WORKDIR /app/api/search
COPY --from=builder /app/api/search/node_modules ./node_modules
COPY . .
RUN mkdir -p /app/api/search/logs && chown -R nodeuser:nodejs /app
USER nodeuser
EXPOSE 8011
CMD ["node", "app.js"]
```

**Line-by-line explanation:**

| Line | What it does |
|---|---|
| `FROM node:18-alpine AS builder` | Starts from an official, lightweight ("alpine") Node.js 18 image. This is stage 1, named `builder`. |
| `WORKDIR /app/api/search` | Sets the "current folder" inside the container. Every subsequent command runs from here. |
| `COPY package*.json ./` | Copies just `package.json` (and `package-lock.json` if it exists) into the image — not the whole codebase yet. This is a caching trick: if dependencies haven't changed, Docker can reuse this layer on the next build. |
| `RUN npm install --omit=dev` | Installs dependencies inside the image, skipping dev-only packages (`--omit=dev`) since they're not needed in production. |
| `FROM node:18-alpine AS runner` | Starts a **fresh, second stage** — this is the actual image that will run in production. Nothing from the builder stage carries over automatically. |
| `RUN addgroup ... && adduser ...` | Creates a non-root Linux user (`nodeuser`) and group (`nodejs`). Running containers as root is a security risk; this avoids it. |
| `COPY --from=builder /app/api/search/node_modules ./node_modules` | Pulls **only** the installed `node_modules` folder from the `builder` stage into this final image — leaving all the build tools and cache behind, keeping the final image smaller. |
| `COPY . .` | Copies the actual application source code into the image. |
| `RUN mkdir -p .../logs && chown -R nodeuser:nodejs /app` | Creates a logs folder and hands ownership of the whole `/app` directory to the `nodeuser` account. |
| `USER nodeuser` | From this point on, the container runs as `nodeuser`, not root. |
| `EXPOSE 8011` | Documents that the app listens on port 8011 (informational — doesn't actually open the port by itself). |
| `CMD ["node", "app.js"]` | The command that runs when the container starts. |

### ⚠️ Critical gotcha: relative import paths

Inside `app.js`, there's this import:
```js
import utils from "../search/common/utils.js";
```
This path only resolves correctly if the container replicates the **exact folder nesting** that exists on the original machine (`api/search/`). That's precisely why `WORKDIR` above is `/app/api/search`, not just `/app` — using the wrong `WORKDIR` produces:
```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/search/common/utils.js'
```

### `.dockerignore`

```
node_modules
.git
.env
.env.*
logs
*.log
Dockerfile
.dockerignore
```
**Explanation:** This file tells Docker which files/folders to **never** copy into the image, even if a `COPY . .` step would otherwise include them. Excluding `.env` is important — secrets shouldn't be baked into an image; they get passed in separately at container-run time instead.

### Build and run commands

```bash
sudo docker build -t global-search-api:local .
```
**Explanation:** `docker build` reads the `Dockerfile` in the current directory (`.`) and produces an image. `-t global-search-api:local` **tags** (names) that image so it can be referenced later — `global-search-api` is the name, `local` is the version tag.

```bash
sudo docker run --rm -d -p 8011:8011 --env-file .env --name search-test global-search-api:local
```
**Explanation, flag by flag:**
- `--rm` — automatically delete the container once it stops (keeps things tidy)
- `-d` — run in "detached" mode, i.e. in the background, freeing up the terminal
- `-p 8011:8011` — maps port 8011 on the host machine to port 8011 inside the container (`host:container`)
- `--env-file .env` — loads environment variables from the `.env` file into the container
- `--name search-test` — gives the running container a friendly name, so it's easy to reference in later commands
- `global-search-api:local` — which image to run

```bash
sudo docker ps
```
**Explanation:** Lists currently running containers — used to confirm the container actually started and is running.

```bash
sudo docker logs search-test
```
**Explanation:** Shows the console output produced by the app inside the container — the first place to look when something goes wrong.

```bash
sudo docker stop search-test
```
**Explanation:** Gracefully stops the named container.

### Common Issue: `npm ci` fails

If `package-lock.json` isn't committed to the repo, this error appears:
```
npm error The `npm ci` command can only install with an existing package-lock.json
```
**Why:** `npm ci` is a stricter install command designed for CI pipelines — it requires an exact lockfile to guarantee reproducible installs, and refuses to run without one.

**Fix:** Use `npm install` instead in the Dockerfile — it's more forgiving and will generate dependencies without requiring a lockfile.

---

## Part C: AWS Setup — IAM & ECR

### What is ECR
**ECR (Elastic Container Registry)** is AWS's private Docker image storage service — functionally similar to Docker Hub, but private to your AWS account. This is where the Kubernetes cluster pulls images from when deploying.

### Understanding IAM permissions

Every AWS resource (like an EC2 instance) has an **IAM Role** attached, which defines exactly what actions it's allowed to perform. New capabilities require **attaching a Policy** to that role — policies are documents that describe permitted actions.

**Policies needed for this workflow:**
- `AmazonEC2ContainerRegistryFullAccess` — required for pushing/pulling Docker images to/from ECR
- `AmazonEC2ReadOnlyAccess` — used at one point to locate a server by name across the AWS account (optional, situational)

**How to attach a policy (AWS Console):**
1. Go to **IAM → Roles**, find the relevant role (e.g. `accord-mf-data-role`)
2. Click **"Add permissions" → "Attach policies"**
3. Search for the policy name, select it, click **"Add permissions"**

### Installing the AWS CLI (if missing)

```bash
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```
**Explanation:** `curl` downloads AWS's official installer package (a zip file) and, via `-o`, saves it locally with the given filename.

```bash
unzip awscliv2.zip
```
**Explanation:** Extracts the contents of the downloaded zip archive.

```bash
sudo ./aws/install
```
**Explanation:** Runs the extracted installer script with administrator privileges (`sudo`), which places the `aws` command in a system-wide location.

```bash
aws --version
```
**Explanation:** Confirms the installation succeeded by printing the installed version number.

> **If an old, `apt`-installed AWS CLI conflicts** (typically shows up as Python `ImportError` tracebacks), remove it first:
> ```bash
> sudo apt remove awscli -y
> hash -r
> ```
> **`hash -r` explained:** Bash caches the file-system location of commands you've already run, for speed. If you delete/replace a command, Bash may still be pointing at the old (now-missing) location. `hash -r` clears this cache, forcing Bash to search the system `PATH` fresh the next time you run a command.

### Confirming your AWS identity

```bash
aws sts get-caller-identity
```
**Explanation:** Asks AWS "who am I, according to my current credentials?" It returns your Account ID and the ARN (Amazon Resource Name) of the IAM User or Role you're authenticated as. If an EC2 instance has an **IAM Role attached**, this works automatically — no manual Access Key/Secret Key setup is needed.

### Creating an ECR repository

```bash
aws ecr describe-repositories --region ap-south-1 --query "repositories[].repositoryName" --output table
```
**Explanation:** Lists existing ECR repositories in the given region. The `--query` flag uses JMESPath syntax to extract just the repository names (rather than the full JSON response), and `--output table` formats it as a readable table instead of raw JSON.

```bash
aws ecr create-repository --repository-name cash-global-search-apis --region ap-south-1
```
**Explanation:** Creates a brand-new ECR repository with the given name. The output includes a `repositoryUri` — a URL-like address needed for pushing images, in the format:
```
<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/cash-global-search-apis
```

### Authenticating Docker with ECR

```bash
aws ecr get-login-password --region ap-south-1 | sudo docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
```
**Explanation:** This is two commands chained with a pipe (`|`):
1. `aws ecr get-login-password` fetches a temporary authentication token from AWS
2. `docker login ... --password-stdin` feeds that token directly into Docker's login process (via "standard input"), rather than typing a password manually — this is safer because the token never gets displayed or stored in shell history

### Tagging and pushing the image

```bash
sudo docker tag global-search-api:local <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/cash-global-search-apis:latest
```
**Explanation:** `docker tag` creates an additional name/label for an existing image, without duplicating any data. This is necessary because Docker requires an image's tag to match the destination registry's address before it can be pushed there.

```bash
sudo docker push <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/cash-global-search-apis:latest
```
**Explanation:** Uploads the tagged image to ECR.

### Verifying

```bash
aws ecr describe-images --repository-name cash-global-search-apis --region ap-south-1
```
**Explanation:** Lists images currently stored in the repository, along with metadata like push date, size, and tags — confirms the push actually succeeded.

---

## Part D: Building and Testing on an EC2 Server

### Why move off the original dev machine
Local/dev VMs can have unreliable networking — in this project, persistent DNS resolution failures made package installation and downloads painfully slow or impossible. Building on an actual EC2 server (AWS's own infrastructure) proved far more reliable.

### Step 1 — SSH key setup, for GitHub access

```bash
ssh-keygen -t ed25519 -C "ec2-deploy-key"
```
**Explanation:** Generates a new SSH key pair. `-t ed25519` specifies a modern, secure key algorithm. `-C` adds a comment (just a label, for identification) to the key. This creates two files: a private key (kept secret, never shared) and a public key (`.pub` — safe to share, this is what gets registered with GitHub).

```bash
cat ~/.ssh/id_ed25519.pub
```
**Explanation:** Prints the **public** key so it can be copied and pasted into GitHub's SSH key settings.

### Step 2 — Test the connection

```bash
ssh -T git@github.com
```
**Explanation:** Attempts an SSH connection to GitHub specifically to test authentication (`-T` disables allocating a full interactive terminal, since GitHub doesn't provide shell access anyway — it just confirms who you are). A successful response looks like:
```
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```

### Step 3 — Clone the repository

```bash
cd ~
git clone git@github.com:Ventura-Securities/project-cash.git
```
**Explanation:** `git clone` downloads a full copy of a remote repository — including its entire history — into a new local folder. The `git@github.com:...` format (as opposed to `https://github.com/...`) tells Git to use SSH for authentication, relying on the key pair set up in Step 1.

```bash
cd project-cash
git checkout develop
```
**Explanation:** `git checkout <branch>` switches your working copy to a different branch. By default, a fresh clone lands on whichever branch is set as the repository's default (often `master`); this switches to `develop`, where active work happens.

### Step 4 onward
Repeat the Dockerfile creation and build/run/test steps from **Part B** on this server.

---

## Part E: The Multi-Architecture Problem and Its Fix

### The problem

After deploying the image to Kubernetes, pods immediately crashed with:
```
exec /usr/local/bin/docker-entrypoint.sh: exec format error
```

### Root cause: CPU architecture mismatch

Every CPU has an **instruction set architecture** — the low-level "language" its processor understands. The two relevant ones here:
- **x86_64 / amd64** — the traditional Intel/AMD architecture, common on most EC2 instances
- **aarch64 / arm64** — the ARM architecture, used by AWS's cost-efficient **Graviton** processors

A Docker image contains **compiled binaries** for one specific architecture. If you build an image on an x86_64 machine and then try to run it on an arm64 machine, the binaries simply cannot execute — hence `exec format error`.

**How this was discovered:**
```bash
# On the build machine
uname -m
```
**Explanation:** `uname -m` prints the machine's hardware architecture. This returned `x86_64`.

```bash
# On the Kubernetes side
kubectl get nodes -o wide
```
**Explanation:** Lists cluster nodes along with extra details (`-o wide`), including the `KERNEL-VERSION` column, which in this case showed `...aarch64` — confirming the EKS worker node is ARM-based (Graviton).

### The fix: multi-platform builds with Docker Buildx

**Buildx** is a Docker extension that can produce images for **multiple architectures from a single build command** — critically, it can even build ARM images on an x86 machine (and vice versa) by using an emulator.

#### Step 1 — Install the Buildx plugin

```bash
mkdir -p ~/.docker/cli-plugins
```
**Explanation:** Docker looks for optional plugins (extra subcommands) in this specific folder. `mkdir -p` creates it, including any missing parent folders, without error if it already exists.

```bash
curl -SL https://github.com/docker/buildx/releases/download/v0.17.1/buildx-v0.17.1.linux-amd64 -o ~/.docker/cli-plugins/docker-buildx
```
**Explanation:** Downloads the official Buildx binary directly from GitHub and saves it under the exact filename (`docker-buildx`) that Docker expects to find in the plugins folder.

```bash
chmod +x ~/.docker/cli-plugins/docker-buildx
```
**Explanation:** `chmod +x` grants "execute" permission on the file. Freshly downloaded files are typically not executable by default — this step is what makes it runnable.

```bash
docker buildx version
```
**Explanation:** Confirms the plugin installed correctly by printing its version.

#### Step 2 — Set up QEMU emulation

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
```
**Explanation:** An x86 machine cannot natively execute ARM instructions. **QEMU** is an emulator that lets it "pretend" to be an ARM machine for the purposes of building a container image. This command:
- Runs a small, pre-built Docker image (`tonistiigi/binfmt`) whose entire job is to register these emulators with the kernel
- `--install all` sets up emulation for every supported foreign architecture (arm64, s390x, etc.), not just the one currently needed
- `--privileged` grants the container deep, low-level access to the host system — required because it's modifying kernel-level binary format handlers
- `--rm` deletes the container as soon as it finishes — it's a one-time setup action, not something that needs to keep running

#### Step 3 — Create a multi-platform builder

```bash
docker buildx create --name multiplatform-builder --use
```
**Explanation:** The default Docker builder can only target the architecture of the machine it's running on. This command creates a **new builder instance** (`docker-container` driver, capable of true multi-platform builds), names it `multiplatform-builder`, and `--use` immediately makes it the active/default builder for all future `docker buildx build` commands.

```bash
docker buildx inspect --bootstrap
```
**Explanation:** `inspect` prints details about the currently active builder — crucially, its `Platforms:` line, which should list both `linux/amd64` and `linux/arm64` if everything is set up correctly. `--bootstrap` also ensures the builder is actually started (not just configured) before inspecting it.

#### Step 4 — Build and push a multi-architecture image

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/cash-global-search-apis:latest \
  --push .
```
**Explanation:**
- `--platform linux/amd64,linux/arm64` — build the image for **both** architectures in one command
- `-t ...` — tag the resulting image with the ECR repository address
- `--push` — upload the result directly to the registry as part of the build. This flag is **mandatory** for multi-platform builds, because a multi-arch image (technically a "manifest list" pointing to two separate images) can't be stored in Docker's local image cache the normal way — it has to go straight to a registry.
- `.` — build context is the current directory (where the Dockerfile lives)

---

## Part F: Manually Deploying to Kubernetes (EKS)

### Kubernetes concepts, explained

| Term | Meaning |
|---|---|
| **Cluster** | The entire Kubernetes system — made up of multiple physical/virtual servers working together |
| **Node** | An individual worker machine within the cluster, where containers actually run |
| **Pod** | The smallest deployable unit in Kubernetes — a group of one or more tightly-coupled containers |
| **Deployment** | A higher-level object that describes "how many Pods, running which image" — Kubernetes continuously works to keep reality matching this description |
| **Service** | Gives a set of Pods a stable network address. Pods can be destroyed and recreated (getting new IPs each time), but a Service's address stays constant |
| **Namespace** | A way of logically partitioning a cluster's resources — e.g., `uatenv` groups together everything belonging to the UAT environment |

### Prerequisites — installing `kubectl`

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
**Explanation:** This is actually two `curl` calls nested together. The inner one (`curl -L -s https://dl.k8s.io/release/stable.txt`) fetches a plain text file containing just the latest stable Kubernetes version number. That result is substituted directly into the outer URL (via `$(...)` command substitution), so the outer `curl -LO` always downloads the current stable `kubectl` binary, without needing to hardcode a version number.

```bash
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
**Explanation:** Makes the downloaded binary executable, then moves it into `/usr/local/bin/` — a directory that's already part of the system's `PATH`, meaning the `kubectl` command becomes available from anywhere in the terminal.

```bash
kubectl version --client
```
**Explanation:** Confirms installation by printing the client tool's version (`--client` avoids trying to also contact a cluster, which wouldn't work yet since it's not configured).

### Configuring cluster access

```bash
aws eks update-kubeconfig --name ventura-mvp-uat-cluster --region ap-south-1
```
**Explanation:** Contacts AWS, retrieves the connection details (API endpoint, certificate, authentication method) for the named EKS cluster, and writes them into `~/.kube/config` — the file `kubectl` reads by default to know which cluster to talk to and how to authenticate.

```bash
kubectl get nodes
```
**Explanation:** Asks the cluster to list its worker nodes — the simplest possible test that the connection and authentication are actually working.

> ⚠️ **A subtlety that caused confusion:** AWS IAM permissions and Kubernetes' own internal permission system (RBAC) are **two separate layers**. Having AWS-level permission to describe/manage the EKS cluster resource does *not* automatically grant permission to run commands *inside* the cluster via `kubectl`. A cluster administrator must explicitly add your IAM identity to the cluster's **Access Entries** (or the older `aws-auth` ConfigMap) before `kubectl` commands will be authorized. In this project, that step was sidestepped entirely by using a pre-existing, already-authorized server (`centralised-monitoring-uat`) instead.

### The Deployment + Service YAML

`global-search-api.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: uatenv
  name: global-search-api
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: global-search-api
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: global-search-api
    spec:
      containers:
        - name: global-search-api
          image: <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/cash-global-search-apis:latest
          ports:
            - containerPort: 8011
          imagePullPolicy: Always
          env:
            - name: ENV
              value: "qa"
          readinessProbe:
            tcpSocket:
              port: 8011
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: 8011
            initialDelaySeconds: 15
            periodSeconds: 20
          startupProbe:
            tcpSocket:
              port: 8011
            failureThreshold: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  namespace: uatenv
  name: global-search-api
spec:
  ports:
    - port: 8011
      targetPort: 8011
      protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: global-search-api
```

**Field-by-field explanation:**

| Field | What it does |
|---|---|
| `apiVersion` / `kind` | Tells Kubernetes what *type* of object this YAML block describes (`Deployment`, `Service`) and which version of that object's schema to use |
| `metadata.namespace` | Which namespace this object belongs to (`uatenv`) |
| `spec.selector.matchLabels` | Tells the Deployment which Pods "belong to it" — must exactly match the `labels` set on the Pod template below |
| `replicas: 1` | How many identical copies of the Pod to keep running |
| `template` | The blueprint used to create each Pod |
| `image:` | The exact image (including registry, repo, and tag) to pull and run |
| `imagePullPolicy: Always` | Forces Kubernetes to pull a fresh copy of the image every time a Pod starts, rather than reusing a possibly-stale local cache |
| `env:` | Injects environment variables directly into the container — used here to set `ENV=qa`, since there's no `.env` file inside the image (it's deliberately excluded via `.dockerignore`) |
| `readinessProbe` | Periodically checks if the app is ready to receive traffic. Until it passes, the Pod won't receive requests from the Service |
| `livenessProbe` | Periodically checks if the app is still functioning. If it fails repeatedly, Kubernetes kills and restarts the container |
| `startupProbe` | Gives a slow-starting app extra grace time before liveness/readiness checks kick in — prevents Kubernetes from prematurely killing a container that just needs more time to boot |
| `type: NodePort` (Service) | Exposes the app on a specific port on every cluster node, making it reachable from outside the cluster |

### Applying the configuration

```bash
kubectl apply -f global-search-api.yaml
```
**Explanation:** Tells Kubernetes "make the cluster's actual state match what's described in this file." If the objects don't exist yet, they're created; if they already exist, they're updated to match.

### Verifying the deployment

```bash
kubectl get pods -n uatenv | grep global-search-api
```
**Explanation:** Lists Pods in the `uatenv` namespace, filtered (`grep`) down to just the ones matching our app's name. The `STATUS` column reveals health at a glance (`Running`, `CrashLoopBackOff`, `ContainerCreating`, etc.).

```bash
kubectl logs <pod-name> -n uatenv
```
**Explanation:** Prints the console output produced inside a specific Pod's container — essential for diagnosing why a Pod is crashing.

```bash
kubectl get svc global-search-api -n uatenv
```
**Explanation:** Shows Service details, including the auto-assigned high-numbered port (e.g. `8011:32116/TCP`) needed to reach the app from outside the cluster when using `NodePort`.

```bash
kubectl get nodes -o wide
```
**Explanation:** Lists nodes along with their internal IP addresses — needed to actually construct a working URL to test the NodePort service.

### Testing via NodePort

```bash
curl "http://<NODE_INTERNAL_IP>:<NODE_PORT>/globalsearch/v1/get?searchkey=reliance&page=0&size=15&type=1"
```
**Explanation:** Same idea as local testing, but pointed at the Node's IP and the auto-assigned NodePort (not the container's internal port `8011`) — this is the actual externally-reachable address for a `NodePort`-type Service.

### Forcing a redeploy after pushing a new image (same tag)

```bash
kubectl rollout restart deployment/global-search-api -n uatenv
```
**Explanation:** If you push a new image using the *same* tag (e.g. `:latest`), Kubernetes has no way to know something changed — from its perspective, the Deployment spec is identical. This command explicitly tells Kubernetes to terminate the existing Pod(s) and create fresh ones, which (combined with `imagePullPolicy: Always`) forces a pull of the newly-pushed image.

---

## Part G: Automating Everything with Jenkins

### Why Jenkins
Everything in Parts B–F was done manually, one command at a time. Jenkins automates this entire sequence — checkout, build, push, deploy — into a single pipeline that runs with one click (or automatically, on every code push).

### Core Jenkins concepts

| Term | Meaning |
|---|---|
| **Controller** | The main Jenkins server — schedules jobs, hosts the web UI, but by default doesn't necessarily have all the build tools installed |
| **Agent (Node)** | A machine (which can be the controller itself, or a separate server) where the actual build steps run |
| **Job / Pipeline** | A defined sequence of steps Jenkins should execute |
| **Jenkinsfile** | A text file, written in Groovy-based syntax, that defines a Pipeline job's steps as code. Critically, **this file lives in the Git repository itself** — Jenkins fetches it fresh from Git on every run, rather than storing it locally on the Jenkins server |

### A critical discovery: matching the right agent to the right tools

Initially, running a build showed:
```
node: not found
docker: not found
```
This happened because the build ran on Jenkins's default execution context, which lacked the required tools. The fix was to identify (and properly label) a specific server that **already had** Docker, Buildx, the AWS CLI, and `kubectl` installed and working — in this case, an EC2 server already running Jenkins itself, named `centralised-monitoring-uat`, with an existing Jenkins node labeled `monitoring-server`.

**Lesson:** rather than installing tools onto Jenkins's default agent, it's often better to point the pipeline at whichever agent already has proven, working tooling — and label that agent clearly so the Jenkinsfile can target it explicitly.

### Setting up the required tools on the agent

Docker, AWS CLI, and `kubectl` were already present on this server. Only Buildx and QEMU emulation needed to be added — using the exact same commands as in **Part E**:

```bash
mkdir -p ~/.docker/cli-plugins
curl -SL https://github.com/docker/buildx/releases/download/v0.17.1/buildx-v0.17.1.linux-amd64 -o ~/.docker/cli-plugins/docker-buildx
chmod +x ~/.docker/cli-plugins/docker-buildx
docker buildx version

docker run --privileged --rm tonistiigi/binfmt --install all

docker buildx create --name multiplatform-builder --use
docker buildx inspect --bootstrap
```

### Creating the Jenkins Job

1. Jenkins Dashboard → **"New Item"**
2. Enter a name (e.g. `Global-search-API`), select type **Pipeline**, click **OK**
3. Under **Triggers**, check **"GitHub hook trigger for GITScm polling"** — this makes Jenkins automatically start a build whenever new code is pushed to GitHub (via a webhook), instead of requiring a manual click every time
4. Under **Pipeline**, configure:

| Field | Value | Why |
|---|---|---|
| Definition | `Pipeline script from SCM` | Tells Jenkins to fetch the Jenkinsfile from a Git repository, rather than using an inline script pasted into the UI |
| SCM | `Git` | The specific source control system being used |
| Repository URL | `https://github.com/Ventura-Securities/project-cash` | Where the code (and the Jenkinsfile) live |
| Credentials | (an existing saved GitHub credential) | Needed because the repo requires authentication |
| Branch Specifier | `*/develop` | Which branch to build from |
| Script Path | `src/api/search/Jenkinsfile` | The exact path, *within the repo*, where Jenkins should look for the Jenkinsfile |

5. **Save**

> ⚠️ **Important:** Because the Jenkinsfile is fetched from Git, it must actually exist at that exact path, on that exact branch, in GitHub — not just on someone's local machine. Attempting to build before the file was pushed produced:
> ```
> ERROR: Unable to find src/api/search/Jenkinsfile from git https://github.com/Ventura-Securities/project-cash
> ```

### Writing and committing the Jenkinsfile

Working from a server that already had the repo cloned and GitHub push access configured:

```bash
cd ~/project-cash
git status
```
**Explanation:** Checks for any uncommitted changes before starting — good practice to avoid accidentally bundling unrelated edits into this commit.

```bash
git pull origin develop
```
**Explanation:** Fetches and merges the latest changes from the remote `develop` branch, to make sure local work starts from the most up-to-date code.

The Jenkinsfile itself (placed at `src/api/search/Jenkinsfile`):

```groovy
pipeline {
    agent {
        label 'monitoring-server'
    }

    environment {
        AWS_REGION   = 'ap-south-1'
        ACCOUNT_ID   = '500535936343'
        ECR_REPO     = 'cash-global-search-apis'
        IMAGE_NAME   = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        K8S_NAMESPACE = 'uatenv'
        DEPLOYMENT_NAME = 'global-search-api'
        APP_DIR      = 'src/api/search'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Login to ECR') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                """
            }
        }

        stage('Build & Push Multi-Arch Image') {
            steps {
                dir("${APP_DIR}") {
                    sh """
                        docker buildx build --platform linux/amd64,linux/arm64 \
                          -t ${IMAGE_NAME}:latest \
                          -t ${IMAGE_NAME}:build-${BUILD_NUMBER} \
                          --push .
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl rollout restart deployment/${DEPLOYMENT_NAME} -n ${K8S_NAMESPACE}
                    kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${K8S_NAMESPACE} --timeout=180s
                """
            }
        }

        stage('Verify') {
            steps {
                sh "kubectl get pods -n ${K8S_NAMESPACE} | grep ${DEPLOYMENT_NAME}"
            }
        }
    }

    post {
        success {
            echo "Deployment successful"
        }
        failure {
            echo "Build failed"
        }
    }
}
```

**Structural explanation:**

| Block | Purpose |
|---|---|
| `pipeline { ... }` | The top-level wrapper for the entire Declarative Pipeline definition |
| `agent { label 'monitoring-server' }` | Forces this entire pipeline to run specifically on the agent(s) carrying this label — guaranteeing it lands on the server with Docker, Buildx, AWS CLI, and `kubectl` already configured |
| `environment { ... }` | Defines reusable variables available throughout every stage, avoiding repetition and centralizing values that might need to change later (e.g., changing regions or repo names in one place) |
| `stage('Checkout') { checkout scm }` | `checkout scm` pulls the exact code from the same repository/branch/commit that this pipeline job is configured against — this is a built-in Jenkins step, not a raw shell command |
| `stage('Login to ECR')` | Runs the same authentication logic as the manual `aws ecr get-login-password \| docker login` sequence from Part C |
| `stage('Build & Push Multi-Arch Image')` | Wraps the build command in `dir("${APP_DIR}")`, which changes the working directory *for the steps inside that block only* (equivalent to `cd`-ing into the folder), since the Dockerfile lives in `src/api/search`, not the repo root. Also tags the image two ways: `:latest` (always points to the newest build) and `:build-<BUILD_NUMBER>` (a unique, permanent tag per build — useful for rollback, since `${BUILD_NUMBER}` is a Jenkins built-in variable that auto-increments) |
| `stage('Deploy to Kubernetes')` | Same `kubectl rollout restart` + `rollout status` sequence as the manual process, with `--timeout=180s` making Jenkins wait up to 3 minutes and **fail the build** if the rollout doesn't succeed in that time — turning a silent failure into a visible one |
| `stage('Verify')` | A final sanity check, printing the Pod's status directly into the build log for anyone reviewing it later |
| `post { success {...} failure {...} }` | Runs regardless of which stages executed — different blocks trigger depending on whether the overall pipeline succeeded or failed, useful for notifications/logging |

### Committing and pushing the Jenkinsfile

```bash
git add Dockerfile .dockerignore Jenkinsfile
```
**Explanation:** Stages exactly these three files for the next commit — deliberately excluding any other locally-modified files (like `.env`) that shouldn't be part of this change.

```bash
git commit -m "Add Dockerfile and Jenkinsfile for automated multi-arch build and K8s deploy"
```
**Explanation:** Creates a permanent snapshot of the staged changes in the repository's history, with a descriptive message explaining the change.

```bash
git push origin develop
```
**Explanation:** Uploads the local commit to the remote GitHub repository's `develop` branch, making it visible to everyone — including Jenkins, which will now be able to find the Jenkinsfile at the configured path.

### Running the pipeline

Back in the Jenkins UI: open the job → **"Build Now"**.

**What happens, stage by stage (as confirmed by an actual successful run):**

1. **Checkout** — Jenkins clones the repo fresh into its own workspace and checks out the exact commit that triggered (or was requested for) the build
2. **Login to ECR** — authenticates Docker against the ECR registry
3. **Build & Push Multi-Arch Image** — Buildx compiles the image for both `linux/amd64` and `linux/arm64` in parallel, then pushes both tagged versions to ECR
4. **Deploy to Kubernetes** — triggers a rolling restart; Kubernetes gracefully terminates the old Pod while bringing up a new one running the freshly-pushed image, with zero downtime
5. **Verify** — confirms the new Pod reached `Running` status

A successful run ends with:
```
Finished: SUCCESS
```

From this point forward, deploying a code change requires nothing more than pushing to `develop` (with the GitHub webhook trigger) or clicking **"Build Now"** — no manual Docker or `kubectl` commands needed.

---

## Common Errors and Fixes (Cheat Sheet)

| Error | Reason | Fix |
|---|---|---|
| `node: not found` / `docker: not found` (Jenkins) | The Jenkins agent executing the build doesn't have these tools installed | Point the pipeline at an agent (via `label`) that already has them, or install the tools on the target agent |
| `Cannot find module '.../utils.js'` (Docker build) | A relative import path depends on the original directory nesting, which the Dockerfile's `WORKDIR` didn't replicate | Set `WORKDIR` to match the correct nested path (e.g. `/app/api/search`) |
| `npm ci` fails — package-lock.json missing | The lockfile isn't committed to the repo | Use `npm install` instead of `npm ci` |
| `ConnectionTimeoutError` (Redis) crashes the app | An unreachable Redis IP throws an unhandled async error | Comment out the connection logic temporarily for local testing; add proper error handling (`.on('error', ...)`) for production |
| Docker `--env-file` — "contains whitespaces" | `.env` uses `KEY = value` (with spaces) | Use `KEY=value` (no spaces around `=`) |
| `port is already allocated` | Another process/container is already using that port | Find and stop it: `sudo lsof -i :<port>` |
| `Temporary failure resolving <host>` | DNS resolution is failing at the OS/network level | Check `resolvectl status` to find which DNS server is actually reachable, and point `/etc/resolv.conf` at it |
| `AccessDeniedException` (AWS CLI) | The IAM Role lacks the specific permission being requested | Attach the relevant IAM Policy via the console |
| `Authentication failed` (Git over HTTPS) | GitHub no longer accepts plain passwords for Git operations | Switch to SSH key authentication, or use a Personal Access Token |
| `Unable to find <path>/Jenkinsfile` | The file exists locally but was never pushed to the branch Jenkins is configured to read | Commit and push the file to the exact branch/path configured in the Jenkins job |
| `exec format error` (inside a container) | The image's CPU architecture doesn't match the machine running it (commonly: built on amd64, run on arm64) | Rebuild using `docker buildx build --platform linux/amd64,linux/arm64 ... --push` |
| `must be logged in to the server` (`kubectl`) | AWS IAM permission exists, but the identity hasn't been granted access *inside* the cluster's own RBAC system | A cluster admin needs to add the identity via EKS Access Entries, or use an already-authorized server/agent |
| `CrashLoopBackOff` (Kubernetes) | The container starts but exits/crashes repeatedly | Run `kubectl logs <pod> -n <namespace>` to see the actual error from inside the container |

---

## Full Commands Reference

### Docker
```bash
docker build -t <name>:<tag> .                              # Build an image from a Dockerfile
docker run -d -p <host>:<container> --name <n> <image>       # Run a container in the background
docker ps                                                     # List running containers
docker ps -a                                                  # List all containers, including stopped ones
docker logs <container>                                       # View a container's console output
docker images                                                 # List locally stored images
docker system df                                              # Show disk space used by Docker
docker image prune -a -f                                      # Remove all unused images (frees disk space)
```

### Docker Buildx (multi-architecture)
```bash
docker buildx version                                         # Confirm buildx is installed
docker buildx create --name <name> --use                      # Create and activate a new multi-platform builder
docker buildx inspect --bootstrap                              # Show builder details and start it if not running
docker buildx build --platform linux/amd64,linux/arm64 -t <tag> --push .   # Build + push for multiple architectures
```

### Git
```bash
git remote -v                    # Show the remote repository's URL
git branch -a                    # List all branches (local and remote)
git checkout <branch>            # Switch to a different branch
git status                       # Show current changes (staged/unstaged/untracked)
git checkout -- <file>           # Discard uncommitted changes to a specific file
git add <file>                   # Stage a file for the next commit
git commit -m "message"          # Create a commit with the staged changes
git push origin <branch>         # Upload local commits to the remote repository
git pull origin <branch>         # Fetch and merge the latest remote changes
```

### AWS CLI
```bash
aws sts get-caller-identity                                       # Show current AWS identity (Account ID, ARN)
aws ecr describe-repositories --region <region>                    # List ECR repositories
aws ecr create-repository --repository-name <name> --region <r>    # Create a new ECR repository
aws ecr get-login-password --region <region>                       # Get a temporary Docker login token for ECR
aws eks list-clusters --region <region>                             # List EKS clusters in the account
aws eks update-kubeconfig --name <cluster> --region <region>        # Configure kubectl to talk to a specific cluster
```

### Kubernetes (`kubectl`)
```bash
kubectl get nodes                                          # List cluster worker nodes
kubectl get nodes -o wide                                   # ...with extra details (IPs, OS, architecture)
kubectl get pods -n <namespace>                              # List Pods in a namespace
kubectl logs <pod> -n <namespace>                            # View a Pod's container logs
kubectl apply -f <file.yaml>                                 # Create/update resources from a YAML file
kubectl get svc -n <namespace>                                # List Services in a namespace
kubectl rollout restart deployment/<name> -n <namespace>      # Force Pods to restart with a fresh image pull
kubectl rollout status deployment/<name> -n <namespace>       # Wait for and report on a rollout's progress
kubectl describe pod <pod> -n <namespace>                     # Detailed diagnostic info about a specific Pod
```

### Networking Diagnostics
```bash
ping -c 3 8.8.8.8              # Test basic internet connectivity (bypasses DNS)
ping -c 3 google.com            # Test DNS resolution + connectivity together
cat /etc/resolv.conf            # View the current DNS server configuration
resolvectl status                # View detailed, per-interface DNS status
sudo lsof -i :<port>             # Find which process is using a specific port
```

---

## Pending / Next Steps

- [ ] Add Vector/ClickHouse-based logging (matching the pattern already used by the `cash-watchlist` service)
- [ ] Commit `package-lock.json` to the repository for fully reproducible builds
- [ ] Add production-grade error handling for Redis connections in `app.js` (the current fix is a local-testing-only workaround)
- [ ] Add explicit CPU/Memory resource limits to the Kubernetes Deployment
- [ ] Increase `replicas` beyond 1 for High Availability once the service is production-ready
- [ ] Review and resolve the dependency vulnerabilities flagged by GitHub Dependabot
- [ ] Consider requiring Pull Requests for changes to `develop`, rather than direct pushes (currently bypassed via admin permissions)

---

*Document compiled from a complete, hands-on debugging and deployment session — 20–21 July 2026.*
