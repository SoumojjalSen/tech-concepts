# Docker Concepts

Reusable concepts for containerised Node.js services, illustrated with the SMC Dockerfile.

## Core model

Docker packages an application and its dependencies into an **image** — a portable, read-only snapshot. Running an image produces a **container** — an isolated process with its own filesystem, network, and process tree.

```text
Dockerfile  ─── docker build ───>  Image  ─── docker run ───>  Container
(recipe)                           (snapshot)                   (running process)
```

- An **image** is built once, stored in a registry (Docker Hub, ECR, GCR), and pulled anywhere.
- A **container** is a running instance of an image. Multiple containers can run from the same image.
- A **registry** stores and distributes images. `docker pull` fetches from a registry; `docker push` uploads to one.

## Images, layers, and caching

An image is a stack of read-only **layers**. Each Dockerfile instruction (`FROM`, `RUN`, `COPY`) creates one layer. Layers are cached — if nothing changed in a layer or its predecessors, Docker reuses the cached version on the next build.

```text
Layer 5:  COPY dist/ → application code
Layer 4:  RUN yarn install → node_modules
Layer 3:  COPY package.json yarn.lock → dependency files
Layer 2:  RUN apt-get install → system packages
Layer 1:  FROM node:20.19.4-slim → base OS + Node.js
```

**Cache invalidation is top-down.** If Layer 3 changes (new dependency in `package.json`), Layers 4 and 5 are rebuilt. If only Layer 5 changes (source code edit), Layers 1–4 are cached.

This is why Dockerfiles copy dependency files first, run install, then copy source code — it avoids re-running `yarn install` on every code change.

## Dockerfile instructions

### FROM

Selects the base image. Every Dockerfile starts with `FROM`.

```dockerfile
FROM node:20.19.4-slim AS builder
```

- `node:20.19.4-slim` — Debian-based image with Node.js pre-installed. `-slim` strips man pages, docs, and extra packages to shrink the image.
- `AS builder` — names this build stage so later stages can copy files from it.

**Tag pinning:** `node:20-slim` is a floating tag — Docker Hub updates it when a new 20.x patch is released. `node:20.19.4-slim` pins to a specific Node version. Pin in production to prevent surprise breakage (e.g., `package.json` `engines.node` rejecting a newer patch).

### WORKDIR

Sets the working directory inside the container. Creates the directory if it does not exist. All subsequent `RUN`, `COPY`, and `CMD` instructions execute relative to this path.

```dockerfile
WORKDIR /app
```

### COPY

Copies files from the build context (your project directory) into the image.

```dockerfile
COPY package.json yarn.lock .puppeteerrc.cjs ./
```

The build context is the directory you pass to `docker build` (the `.` in `docker build -t smc-test .`). Files listed in `.dockerignore` are excluded — they never enter the build context.

### ARG and ENV

Both set variables, but at different scopes:

| | ARG | ENV |
|---|---|---|
| Availability | Build time only (during `docker build`) | Build time AND runtime (inside the container) |
| Set from CLI | `--build-arg APP_ENV=production` | `--env APP_ENV=production` or `-e` |
| Persisted in image | No — disappears after build | Yes — baked into the image metadata |

```dockerfile
ARG APP_ENV                                     # build-time only
ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium  # available at runtime
```

### RUN

Executes a shell command inside the image and commits the result as a new layer. Used for installing packages, compiling code, or any build-time setup.

```dockerfile
RUN apt-get update \
  && apt-get install -y --no-install-recommends chromium tini wget \
  && rm -rf /var/lib/apt/lists/*
```

Chain commands with `&&` in a single `RUN` to produce one layer instead of three. The `rm -rf /var/lib/apt/lists/*` at the end deletes the apt cache — it served its purpose during install and would otherwise bloat the image.

### COPY --from (multi-stage builds)

Copies files from a named build stage rather than from the build context.

```dockerfile
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
```

The `builder` stage compiled TypeScript and installed dependencies. The `runtime` stage only takes the compiled output and production `node_modules` — no source code, no devDependencies, no build tools. This is a **multi-stage build** and is the standard pattern for small, secure production images.

### USER

Switches all subsequent instructions (and the container's runtime process) to a non-root user.

```dockerfile
RUN groupadd --gid 1001 nodejs \
  && useradd --uid 1001 --gid nodejs --create-home --shell /usr/sbin/nologin nodejs

USER nodejs
```

Running as root inside a container is a security risk. If an attacker exploits the application, they inherit root's permissions inside the container. A non-root user limits the blast radius.

### EXPOSE

Documents which port the container listens on. It does not actually publish the port — that requires `-p` at `docker run` time.

```dockerfile
EXPOSE 3000
```

`EXPOSE` is metadata for humans and orchestrators (Kubernetes, Docker Compose). The application still binds to port 3000 regardless of whether `EXPOSE` is present.

### ENTRYPOINT and CMD

Together they define the command that runs when the container starts.

```dockerfile
ENTRYPOINT [ "/usr/bin/tini", "-g", "--", "/app/scripts/docker-entrypoint.sh"]
CMD ["sh", "scripts/start.sh"]
```

- **ENTRYPOINT** is the fixed part of the command — it always runs. Here it starts `tini` (the init process) which then runs `docker-entrypoint.sh`.
- **CMD** provides default arguments appended to ENTRYPOINT. It can be overridden at `docker run` time.

The effective command is: `tini -g -- /app/scripts/docker-entrypoint.sh sh scripts/start.sh`.

`docker-entrypoint.sh` sets up environment variables for on-demand environments, then calls `exec "$@"` which replaces the shell with whatever CMD passed (`sh scripts/start.sh`). `exec` is important — it makes the Node process a direct child of tini, so signals route correctly.

### HEALTHCHECK

Tells Docker how to check if the container is healthy.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
```

| Flag | Meaning |
|---|---|
| `--interval=30s` | Check every 30 seconds |
| `--timeout=3s` | Fail the check if no response within 3 seconds |
| `--start-period=5s` | Grace period after container start (failures during this window do not count) |
| `--retries=3` | Mark unhealthy after 3 consecutive failures |

Docker marks the container as `healthy`, `unhealthy`, or `starting`. Orchestrators (Kubernetes, ECS) use this to decide whether to route traffic to the container or restart it.

## Multi-stage builds

A multi-stage Dockerfile has multiple `FROM` instructions. Each one starts a new stage with a fresh filesystem. The final image contains only the last stage.

```text
Stage 1 (builder):                    Stage 2 (runtime):
  FROM node:20.19.4-slim                FROM node:20.19.4-slim
  COPY source code                      apt-get install chromium, tini
  yarn install (all deps)               COPY --from=builder dist/
  yarn build                            COPY --from=builder node_modules/
  npm prune --omit=dev                  (no source, no devDeps, no build tools)
```

**Why:** A build stage needs TypeScript, ESLint, webpack, and all devDependencies. The runtime stage needs none of that. Separating them produces a smaller, more secure image — fewer packages means fewer potential vulnerabilities and a faster pull from the registry.

## .dockerignore

The `.dockerignore` file excludes files from the **build context** — the directory Docker sends to the daemon when you run `docker build`. Without it, Docker would send everything (including `node_modules`, `.git`, `.env` files, IDE configs) to the daemon, making builds slower and potentially leaking secrets into the image.

```text
node_modules       ← the image installs its own; host node_modules is wrong platform anyway
dist               ← the image builds its own
.git               ← not needed in the image, and it is large
.env               ← secrets must not be baked into the image
.env.*
thoughts           ← internal docs, not needed at runtime
```

The `!` prefix is an exception — `!docs/openapi.yml` means "include this file even though `docs/` is excluded."

## tini — the init process

Every Linux system has PID 1, the init process. In a Docker container, the first process you start becomes PID 1. Normal applications are not designed to be PID 1 — they do not:

1. **Reap zombie processes** — child processes that exit but whose parent never called `wait()` accumulate as zombies.
2. **Forward signals** — the kernel delivers SIGTERM/SIGINT to PID 1, but PID 1 does not forward them to children by default.

`tini` is a tiny init process (20 KB) that solves both problems. It sits as PID 1, forwards signals to the application, and reaps zombies.

```text
Without tini:                      With tini:
  PID 1: node dist/index.js          PID 1: tini
    (misses SIGTERM,                    PID 2: node dist/index.js
     zombies accumulate)                  (receives SIGTERM, drains cleanly)
```

The `-g` flag in `tini -g --` means "forward signals to the entire process group," so child processes spawned by Node (like Chromium for Puppeteer) also receive the signal.

## docker-entrypoint.sh and exec

The entrypoint script runs before the main application. In SMC it handles on-demand environment setup — exporting environment-specific URLs, database hosts, Kafka topics, and Temporal config when `ENVIRONMENT=onDemand`.

The script ends with:

```bash
exec "$@"
```

`exec` replaces the current shell process with the command passed as arguments (from CMD). Without `exec`, the shell would remain as a parent process between tini and node — signals would go to the shell instead of node, and the shell would mask node's exit code. With `exec`, node becomes tini's direct child.

```text
Without exec:                       With exec:
  tini → sh (entrypoint.sh)           tini → node dist/index.js
           → node dist/index.js        (direct child, clean signal path)
  (extra shell layer)
```

## Build cache mounts

```dockerfile
RUN --mount=type=cache,target=/root/.cache/yarn,sharing=locked \
    yarn install --frozen-lockfile
```

A cache mount persists a directory across builds without baking it into the image layer. Here, yarn's download cache survives between builds — if you rebuild after adding one new package, yarn only downloads that package instead of all 500+.

`sharing=locked` means concurrent builds wait for the lock rather than running simultaneously against the same cache (prevents corruption).

## Essential commands

### Building

```bash
docker build -t smc-test .                        # build from Dockerfile in current directory
docker build -t smc-test --build-arg APP_ENV=dev . # pass a build argument
docker build --no-cache -t smc-test .              # force rebuild (ignore cache)
docker build --target builder -t smc-debug .       # build only up to the "builder" stage
```

### Running

```bash
docker run smc-test                                # start a container (foreground)
docker run -d smc-test                             # start in background (detached)
docker run -p 3000:3000 smc-test                   # map host port 3000 → container port 3000
docker run -e ENVIRONMENT=development smc-test     # set an environment variable
docker run --name smc smc-test                     # give the container a name
docker run -d -p 3000:3000 --name smc smc-test     # all together
```

`-p host:container` is required to access the container from your machine. Without it, the container's port 3000 is isolated.

### Inspecting

```bash
docker ps                      # list running containers
docker ps -a                   # list all containers (including stopped)
docker logs smc                # view container stdout/stderr
docker logs -f smc             # follow logs in real time (like tail -f)
docker inspect smc             # full container metadata (JSON)
docker exec -it smc sh         # open a shell inside a running container
docker exec smc ls /app/dist   # run a one-off command inside the container
```

`-it` means interactive + allocate a TTY — required for an interactive shell session.

### Stopping and cleanup

```bash
docker stop smc                # send SIGTERM, wait 10s, then SIGKILL
docker stop -t 30 smc          # send SIGTERM, wait 30s, then SIGKILL
docker rm smc                  # remove a stopped container
docker rmi smc-test            # remove an image
docker system prune            # remove all stopped containers, unused images, build cache
docker system prune -a         # remove everything not currently in use (aggressive)
```

`docker stop` sends SIGTERM first — this is what triggers tini → node → graceful shutdown (drain Temporal worker, close DB connections). After the timeout, Docker force-kills with SIGKILL.

### Images

```bash
docker images                  # list local images
docker image ls                # same as above
docker history smc-test        # show layers and their sizes
docker image inspect smc-test  # full image metadata
```

## Volumes and bind mounts

Containers have an ephemeral filesystem — when the container stops, its data is gone. **Volumes** persist data beyond the container's lifecycle.

```bash
docker run -v mydata:/app/data smc-test            # named volume (Docker manages storage)
docker run -v $(pwd)/logs:/app/logs smc-test       # bind mount (host directory → container)
```

| Type | Use case |
|---|---|
| Named volume | Database data, persistent state |
| Bind mount | Development: mount source code so changes appear inside the container without rebuilding |

For local Temporal testing, the SQLite file can be bind-mounted:

```bash
mkdir -p .temporal
docker run -v $(pwd)/.temporal:/data temporal-server --db-filename /data/temporal.db
```

## Networking

By default, each container gets its own network namespace. Containers can talk to each other via Docker networks.

```bash
docker network create smc-net                      # create a network
docker run --network smc-net --name temporal ...   # attach Temporal to the network
docker run --network smc-net --name smc ...        # attach SMC to the same network
```

On a shared network, containers reach each other by name: SMC can connect to `temporal:7233` because Docker's built-in DNS resolves the container name `temporal` to its IP on the `smc-net` network.

`host.docker.internal` is a special DNS name (macOS/Windows) that resolves to the host machine. Use it when a container needs to reach a service running on the host (not in Docker).

## Docker Compose

Docker Compose defines multi-container applications in a single `docker-compose.yml` file. Instead of running multiple `docker run` commands with networks, volumes, and environment variables, you declare everything in YAML and run `docker compose up`.

```yaml
services:
  temporal:
    image: temporalio/auto-setup:latest
    ports:
      - "7233:7233"     # gRPC
      - "8233:8233"     # Web UI
    environment:
      - DB=sqlite
    volumes:
      - .temporal:/etc/temporal/data

  smc:
    build: .
    ports:
      - "3000:3000"
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - ENVIRONMENT=development
    depends_on:
      - temporal
```

```bash
docker compose up              # start all services (foreground)
docker compose up -d           # start in background
docker compose logs -f smc     # follow logs for one service
docker compose down            # stop and remove everything
docker compose build           # rebuild images
docker compose ps              # list running services
```

`depends_on` controls startup order (temporal starts before smc) but does not wait for temporal to be healthy — use `depends_on.condition: service_healthy` with a healthcheck for that.

## Container Registries (Pull / Push)

A **registry** is where Docker images are stored and distributed. Like GitHub for code, but for Docker images.

| Registry | URL format | Free? |
|----------|-----------|-------|
| Docker Hub | `docker.io/username/image` | Yes (public repos) |
| GitHub Container Registry | `ghcr.io/username/image` | Yes (public repos) |
| Oracle OCIR | `region.ocir.io/namespace/image` | Yes (Oracle free tier) |
| AWS ECR | `account.dkr.ecr.region.amazonaws.com/image` | Paid |

### Pushing an image

```bash
# 1. Build the image
docker build -t ghcr.io/soumojjalsen/flowpilot:latest .

# 2. Log in to the registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# 3. Push
docker push ghcr.io/soumojjalsen/flowpilot:latest
```

### Pulling an image

```bash
# Pull from registry → saves locally
docker pull ghcr.io/soumojjalsen/flowpilot:latest

# Run it (auto-pulls if not found locally)
docker run ghcr.io/soumojjalsen/flowpilot:latest
```

### Tags

Tags version your images. Common patterns:

```bash
ghcr.io/user/app:latest      # floating — always points to newest
ghcr.io/user/app:v1.2.3      # semver — pinned release
ghcr.io/user/app:abc123def   # git commit SHA — exact build
```

`latest` is convenient but dangerous in production — it changes under you. Pin to a SHA or version for reproducibility.

### CI/CD: GitHub Actions → Registry

```
Push to main → GitHub Actions builds image → pushes to ghcr.io
                                                    │
Your VM: docker pull ghcr.io/user/app:latest ←──────┘
                                                    │
                                              runs the container
```

`GITHUB_TOKEN` is provided automatically in GitHub Actions — no extra secrets to configure for ghcr.io.

## Multi-Arch Builds

Your Mac has an Apple Silicon (ARM) chip. Your Oracle VM has an ARM CPU. But CI runners are x86 (amd64). A multi-arch image works on all of them.

```bash
# Build for both architectures
docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/user/app:latest --push .
```

Docker automatically pulls the right architecture when you `docker pull`. The image manifest contains both variants — one binary, works everywhere.

In GitHub Actions, `docker/setup-qemu-action` enables cross-platform builds (emulates ARM on the x86 CI runner).

## Environment Variables and Secrets

Three ways to pass env vars to a container:

```bash
# 1. Inline (visible in shell history + docker inspect)
docker run -e API_KEY=secret123 myapp

# 2. From host environment (better — not in command)
export API_KEY=secret123
docker run -e API_KEY myapp

# 3. From .env file (best — secrets in a file with chmod 600)
docker run --env-file .env myapp
```

`.env` file format:
```
API_KEY=secret123
DATABASE_URL=postgres://localhost/mydb
PORT=3000
```

Always use `--env-file` for secrets. Never `-e` with the value inline.

## Restart Policies

What happens when a container crashes or the host reboots:

| Policy | Behavior |
|--------|----------|
| `no` (default) | Container stays stopped |
| `always` | Always restart — on crash, on reboot, even if manually stopped then host reboots |
| `unless-stopped` | Like `always` but respects manual `docker stop` |
| `on-failure` | Restart only on non-zero exit code, not on reboot |

```bash
docker run --restart always myapp    # survives crashes + reboots
docker run --restart unless-stopped myapp  # survives crashes + reboots, respects manual stop
```

For server deployments (CLIProxyAPI, FlowPilot, n8n), use `--restart always` or `unless-stopped`.

## Real-World Docker Session (FlowPilot + Oracle VM)

A complete Docker session for deploying services on an Oracle ARM VM:

### Installing Docker on a fresh VM

```bash
sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker ubuntu    # add user to docker group (needs re-login)
sudo docker run --rm hello-world  # verify it works (--rm cleans up after)
```

### Pulling and running a third-party service

```bash
# Pull image from Docker Hub
sudo docker pull eceasy/cli-proxy-api:latest

# Run it as a background service
sudo docker run -d --name cliproxyapi --restart always \
  -p 8317:8317 \
  -v ~/app/cliproxyapi/config.yaml:/CLIProxyAPI/config.yaml \
  -v ~/app/cliproxyapi/auths:/root/.cli-proxy-api \
  eceasy/cli-proxy-api:latest
```

Breakdown of this command:
- `-d` → background
- `--name cliproxyapi` → name for `docker stop/logs/exec`
- `--restart always` → survives crashes + reboots
- `-p 8317:8317` → expose port
- `-v host:container` → mount config file and auth directory from VM into container
- Last arg is the image

### Inspecting a running container

```bash
sudo docker logs cliproxyapi              # view output
sudo docker logs cliproxyapi | grep "client"  # filter logs
sudo docker restart cliproxyapi           # restart (picks up new config)
sudo docker exec -it cliproxyapi /bin/sh  # open shell inside container
```

### Running your own app from GitHub Container Registry

```bash
# Pull from ghcr.io (built by GitHub Actions CI)
sudo docker pull ghcr.io/soumojjalsen/flowpilot:latest

# Run with secrets from .env file
sudo docker run -d --name flowpilot --restart always \
  -p 3000:3000 \
  --env-file ~/app/flowpilot/.env \
  ghcr.io/soumojjalsen/flowpilot:latest
```

### Running n8n (workflow automation)

```bash
docker run -d --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n
```

`-v n8n_data:/home/node/.n8n` is a **named volume** — n8n stores workflows and credentials there. Survives `docker stop/rm`. Without it, deleting the container loses all workflows.

### Interactive auth inside a container

```bash
# Open shell inside running container
sudo docker exec -it cliproxyapi /bin/sh

# Inside the container, run the OAuth login
/CLIProxyAPI/CLIProxyAPI -claude-login -no-browser
# Prints a URL → open in browser → paste code back
```

`docker exec` runs a command in an already-running container. `-it` makes it interactive (you can type). Without `-it`, one-off commands work too:

```bash
sudo docker exec cliproxyapi ls /CLIProxyAPI/   # non-interactive, just prints output
```

### Complete flag reference

| Flag | Meaning |
|------|---------|
| `run` | Create + start a new container |
| `-d` | Detached (background) |
| `--name X` | Name the container |
| `--restart always` | Auto-restart on crash or VM reboot |
| `-p host:container` | Map a port |
| `-v host:container` | Mount a file or directory |
| `-v name:container` | Named volume (Docker manages storage location) |
| `--rm` | Delete container when it stops (for one-off tests) |
| `--env-file .env` | Load env vars from file |
| `-e KEY=VAL` | Set one env var inline |
| `-it` | Interactive + allocate TTY (for shells) |
| `pull` | Download image from registry |
| `logs` | View container stdout/stderr |
| `logs -f` | Follow logs in real time |
| `restart` | Stop + start a container |
| `exec` | Run command inside running container |
| `stop` | Send SIGTERM, then SIGKILL after timeout |
| `rm` | Remove a stopped container |
| `rmi` | Remove an image |
| `ps` | List running containers |
| `ps -a` | List all containers (including stopped) |

---

## Image size and security

Smaller images are faster to pull, faster to deploy, and have fewer packages that could contain vulnerabilities.

| Technique | How it helps |
|---|---|
| Multi-stage build | Final image has no source code, no devDependencies, no build tools |
| `-slim` base images | Strips docs, man pages, extra utilities |
| `npm prune --omit=dev` | Removes devDependencies from node_modules |
| `rm -rf /var/lib/apt/lists/*` | Removes apt cache after install |
| `.dockerignore` | Prevents unnecessary files from entering the build context |
| Non-root user | Limits damage if the application is compromised |
| Pinned base image tags | Prevents unexpected version changes from introducing vulnerabilities |

## engines.node in package.json

```json
{
  "engines": {
    "node": "20.19.4"
  }
}
```

This field tells package managers (yarn, npm) to reject installs if the running Node version does not match. With an exact version like `"20.19.4"`, floating Docker tags like `node:20-slim` can break the build when Docker Hub updates to a newer patch (e.g., 20.20.2). The fix is to pin the Dockerfile's `FROM` tag to the same version: `node:20.19.4-slim`.

## Signal flow in a container

Understanding signal flow explains how graceful shutdown works end-to-end:

```text
docker stop  →  SIGTERM  →  tini (PID 1)  →  node (PID 2)
                                                 ↓
                                            index.ts signal handler
                                                 ↓
                                            stopTemporalRuntime()
                                            (drain worker, close connections)
                                                 ↓
                                            process.exit(0)
                                                 ↓
                                            container exits cleanly
```

If the process does not exit within the stop timeout (default 10 seconds, configurable with `-t`), Docker sends SIGKILL — an immediate, non-catchable kill. This is why `shutdownGraceTime` on the Temporal worker (20s) should be less than or equal to the `docker stop` timeout, and the Kubernetes `terminationGracePeriodSeconds` should accommodate both the drain time and any cleanup after it.

## SSH tunnels and port forwarding

**Port forwarding** = "when I hit port X on my machine, send the traffic to port Y on a remote server."
**SSH tunnel** = the encrypted pipe that carries that traffic. They're the same thing in practice.

```text
Your Mac                          Remote VM
┌──────────┐     SSH tunnel      ┌──────────┐
│ :5678 ───┼────encrypted────────┼─→ :5678  │ (n8n)
│ :3000 ───┼────encrypted────────┼─→ :3000  │ (ai-toolbox)
│ :8317 ───┼────encrypted────────┼─→ :8317  │ (CLIProxyAPI)
└──────────┘     (internet)      └──────────┘
```

```bash
ssh -i ~/.ssh/key -L 5678:localhost:5678 -L 3000:localhost:3000 -N -f user@vm-ip
```

| Flag | Meaning |
|------|---------|
| `-i ~/.ssh/key` | Private key for auth |
| `-L 5678:localhost:5678` | Forward local port 5678 → remote localhost:5678 |
| `-N` | Don't open a shell — just keep tunnels open |
| `-f` | Run in background |

Multiple `-L` flags forward multiple ports in one command. Without tunnels, services on the VM aren't reachable from your laptop (ports aren't open to internet). Once you set up a reverse proxy (Caddy/nginx) with a public domain, tunnels become unnecessary.

`-L` stands for **Local** port forwarding. There's also `-R` (Remote) which does the reverse — forwards a remote port to your local machine.

### Managing tunnels

```bash
# List running tunnels
ps aux | grep "ssh.*-L.*oracle_key" | grep -v grep

# Kill a specific tunnel by PID
kill <PID>

# Kill all tunnels using oracle_key
pkill -f "ssh.*-L.*oracle_key"

# Check what's using a port
lsof -i :5678
```

### SSH multiplexing

When you already have an SSH session open to the VM and start a tunnel, SSH detects "I already have a connection to this server" and tries to reuse it. If it can't, it shows:

```
ControlSocket ... already exists, disabling multiplexing
```

This is not an error — it just opened a separate connection. The tunnel works fine either way. Multiplexing is SSH being efficient by sharing connections.

## Docker Compose

Docker Compose runs **multiple containers** from a single config file. Instead of writing separate `docker run` commands for each service, you define everything in `docker-compose.yml`.

### Without compose (manual)

```bash
docker run -d --name cliproxyapi --network host eceasy/cli-proxy-api:latest
docker run -d --name ai-toolbox --network host --env-file /etc/ai-toolbox/.env ghcr.io/soumojjalsen/ai-toolbox:latest
docker run -d --name n8n --network host -v n8n_data:/home/node/.n8n n8nio/n8n
```

### With compose (one command)

```yaml
name: app-server

services:
  cliproxyapi:
    image: eceasy/cli-proxy-api:latest
    container_name: cliproxyapi
    network_mode: host
    volumes:
      - cliproxyapi_data:/CLIProxyAPI
    restart: unless-stopped

  ai-toolbox:
    image: ghcr.io/soumojjalsen/ai-toolbox:latest
    container_name: ai-toolbox
    network_mode: host
    env_file: /etc/ai-toolbox/.env
    restart: unless-stopped
    depends_on:
      - cliproxyapi

  n8n:
    image: n8nio/n8n
    container_name: n8n
    network_mode: host
    volumes:
      - n8n_data:/home/node/.n8n
    restart: unless-stopped

volumes:
  n8n_data:
  cliproxyapi_data:
```

```bash
docker-compose up -d       # start all
docker-compose ps          # status
docker-compose logs n8n    # logs for one service
docker-compose down        # stop and remove all
docker-compose up -d n8n   # start only one service
docker-compose pull        # pull latest images
```

### Key fields

| Field | Meaning |
|-------|---------|
| `name` | Project name — prefixed to container/volume names |
| `image` | Docker image to pull and run |
| `container_name` | Explicit container name (otherwise compose auto-generates one) |
| `network_mode: host` | Container shares the host's network — no port mapping needed, all services reach each other via localhost |
| `env_file` | Load environment variables from a file |
| `depends_on` | Start this service after the listed service. Controls startup order only, not "wait until healthy" |
| `volumes` | Mount persistent storage (see Volumes section below) |
| `restart: unless-stopped` | Auto-restart on crash or VM reboot, unless manually stopped |

### `docker compose` (space) vs `docker-compose` (hyphen)

- `docker compose` = plugin, installed via `docker-compose-plugin` package
- `docker-compose` = standalone binary, installed separately

Same functionality, different install method. Use whichever is installed.

## Volumes

A container's filesystem is **temporary** — `docker rm` deletes everything inside. A **volume** is permanent storage on the host disk that survives container removal.

```yaml
volumes:
  - cliproxyapi_data:/CLIProxyAPI
#   ↑ volume name      ↑ path inside container
```

**Left side** = named volume on the host disk (persistent).
**Right side** = folder inside the container where it's mounted.

Everything written to `/CLIProxyAPI/` inside the container is actually saved in the volume. Next container that mounts the same volume sees the same files.

### Analogy: USB drive

```text
Without volume:
  Container writes to /data → container removed → data gone

With volume:
  Container writes to /data → saved to USB (volume)
  Container removed → USB still has the data
  New container mounts USB at /data → data is back
```

### Where volumes live on disk

```bash
docker volume ls                              # list all volumes
docker volume inspect cliproxyapi_data        # see physical path
# Usually: /var/lib/docker/volumes/<name>/_data/
```

### Volume naming with compose

Compose prefixes volume names with the project name:

```
docker-compose.yml: name: app-server, volume: n8n_data
Actual volume name: app-server_n8n_data
```

### Declaring volumes

Volumes must be declared at the bottom of the compose file:

```yaml
volumes:
  n8n_data:           # declares the volume
  cliproxyapi_data:   # declares the volume
```

Without declaration, compose treats the left side as a host directory path instead of a named volume.

### Host file mount vs named volume

Two types of volume mounts in compose:

```yaml
volumes:
  - ./Caddyfile:/etc/caddy/Caddyfile   # host file mount
  - caddy_data:/data                     # named volume
```

| Type | Syntax | How it works | Use case |
|------|--------|-------------|----------|
| **Host file mount** | Starts with `./` or `/` | Maps a specific file/folder from the host to the container. You edit on host, container sees the change. | Config files you want to version control (Caddyfile, nginx.conf) |
| **Named volume** | Just a name (no path prefix) | Docker manages the storage. Persists across container removal. You don't edit files in it directly. | Data that the app generates (databases, auth tokens, certificates) |

Host file mounts require the file to exist on the host **before** the container starts. Named volumes are created automatically by Docker.

## Caddyfile

Caddy's routing configuration. Defines how incoming requests are routed to backend services.

```
:80 {                              # Listen on port 80 (HTTP)

    handle /api/* {                # If URL starts with /api/
        uri strip_prefix /api      # Remove /api from the path
        reverse_proxy localhost:3000  # Forward to ai-toolbox
    }
    # /api/health → strips /api → sends /health to localhost:3000

    handle /workflow/* {           # If URL starts with /workflow/
        uri strip_prefix /workflow # Remove /workflow from the path
        reverse_proxy localhost:5678  # Forward to n8n
    }
    # /workflow/login → strips /workflow → sends /login to localhost:5678

    handle /health {               # Exact match /health
        respond "ok" 200           # Caddy replies directly (no forwarding)
    }

    handle {                       # Catch-all (everything else)
        respond "app-server" 200
    }
}
```

| Directive | Meaning |
|-----------|---------|
| `:80` | Listen on port 80 |
| `handle /path/*` | Match URLs starting with /path/ |
| `uri strip_prefix /path` | Remove /path from the URL before forwarding |
| `reverse_proxy localhost:3000` | Forward the request to a backend service |
| `respond "text" 200` | Caddy replies directly without forwarding |

### Why strip_prefix?

The prefix (e.g. `/ai-toolbox`) is Caddy's routing concern — the backend service doesn't know about it. Without stripping, the backend receives a path it doesn't recognize.

```
Without strip_prefix:
  Browser: /ai-toolbox/health
  Caddy forwards: /ai-toolbox/health → ai-toolbox
  ai-toolbox: "no route /ai-toolbox/health" → 404

With strip_prefix:
  Browser: /ai-toolbox/health
  Caddy strips /ai-toolbox → forwards: /health → ai-toolbox
  ai-toolbox: "I know /health" → 200 ok
```

This is standard practice in every reverse proxy (nginx does the same with `proxy_pass` trailing slash). The backend code stays clean — it defines `/health`, `/ai`, `/mcp/:server/:tool` without knowing what prefix Caddy puts in front.

### External vs internal access

```
From internet (through Caddy):
  http://server-ip/ai-toolbox/health → Caddy → strip prefix → localhost:3000/health

From inside VM (direct, no Caddy):
  http://localhost:3000/health → ai-toolbox directly
```

Services on the same VM talk to each other via `localhost` directly — they don't go through Caddy. Only external traffic from the internet goes through Caddy.

### Validating a Caddyfile

```bash
docker run --rm -v ./Caddyfile:/etc/caddy/Caddyfile caddy:2-alpine caddy validate --config /etc/caddy/Caddyfile
```

Run locally before deploying to catch syntax errors.

## docker-compose run

`docker-compose run` creates a **one-off** container from a service to run a specific command, instead of the default startup command.

```bash
docker-compose run --rm cliproxyapi ./CLIProxyAPI -claude-login
```

| Part | Meaning |
|------|---------|
| `run` | One-off container, not a long-running service |
| `--rm` | Remove the container when it exits (cleanup) |
| `cliproxyapi` | Service name from the compose file |
| `./CLIProxyAPI -claude-login` | Override the default command |

The container gets the same image, volumes, and network as the service definition. So files written during `run` (like `config.yaml` in a volume) are available when the service starts normally with `up -d`.

## Infra repo pattern

Industry standard: separate "what the app does" from "how it runs."

| Repo | Contains | Purpose |
|------|----------|---------|
| App repo (e.g. `ai-toolbox`) | Source code, Dockerfile, CI workflow | How to **build** the image |
| Infra repo (e.g. `server-config`) | docker-compose.yml, Caddyfile, deploy workflow | How to **run** the images |

The app repo owns its Dockerfile. The infra repo owns the compose/K8s manifests. This mirrors GitOps tools like ArgoCD where deployment config is always in a separate repo from app code.

## Env files in production

| Location | Purpose |
|----------|---------|
| `.env` in project root | Local dev — loaded by `node --env-file=.env` or `docker-compose` |
| `.env.example` in repo | Committed — documents required vars with placeholder values |
| `/etc/<service>/.env` on server | Production — `chmod 600` for security, referenced by compose `env_file:` |

Docker compose loads env files via `env_file:` field. Multiple `--env-file` flags in `docker run` merge values (later file wins on duplicates).

## Auto-deploy with GitHub Actions

```yaml
name: Deploy to VM
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VM_HOST }}
          username: ${{ secrets.VM_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd ~/apps/server-config
            docker-compose down --remove-orphans || true
            docker-compose pull
            docker-compose up -d
```

Push to the infra repo → GitHub Actions SSHs into the VM → pulls latest images → restarts services. Secrets (SSH key, host IP) stored in GitHub repo settings, never in code.

`docker-compose down --remove-orphans || true` — stops/removes old containers before starting new ones. `|| true` ignores errors on first run when no containers exist yet.

## docker-compose exec vs run

| | `exec` | `run` |
|---|--------|-------|
| Target | Already-running container | Creates a new temporary container |
| Use case | One-off command inside a live service | Run setup/migration before service starts |
| State | Changes persist in the running container | With `--rm`, container and its writes are deleted on exit |
| Example | `docker-compose exec cliproxyapi ./CLIProxyAPI -claude-login` | `docker-compose run --rm cliproxyapi sh -c "cat > /CLIProxyAPI/config.yaml ..."` |

Key difference for auth tokens: if you save credentials via `run --rm`, the temporary container is deleted and the tokens vanish. Use `exec` to write into the running container, or mount a volume so the directory persists regardless.

```bash
# exec — runs inside the live container
docker-compose exec ai-toolbox npx -y mcp-remote@0.1.38 https://mcp.groww.in/mcp 52155

# run — creates a fresh container, removes it after
docker-compose run --rm cliproxyapi ./CLIProxyAPI -claude-login
```

## docker cp

Copies files between the host and a container.

```bash
# Host → Container
docker cp ~/.mcp-auth/. ai-toolbox:/root/.mcp-auth/

# Container → Host
docker cp ai-toolbox:/root/.mcp-auth/. ~/backup-mcp-auth/
```

The trailing `/.` copies the **contents** of the directory, not the directory itself. Useful for seeding a volume or extracting files for debugging.

## OAuth token persistence in Docker

Tokens saved inside a container are **ephemeral** — lost on container restart, removal, or image update. Mount the token directory to a named volume to persist them.

**Common pitfall:** the volume is mounted at the wrong path. The app may store config and auth in different directories:

```text
CLIProxyAPI example:
  /CLIProxyAPI/config.yaml        ← config (port, api_key)
  /root/.cli-proxy-api/auth.json  ← OAuth tokens (Claude login)

These are different directories. A volume at /CLIProxyAPI/ does NOT capture /root/.cli-proxy-api/.
```

Fix: mount a separate volume for each:

```yaml
volumes:
  - cliproxyapi_data:/CLIProxyAPI           # config
  - cliproxyapi_auth:/root/.cli-proxy-api   # Claude OAuth tokens
  - mcp_auth:/root/.mcp-auth               # Groww/Kite OAuth tokens (mcp-remote)
```

After mounting, authenticate **once** — the volume persists the tokens across all future restarts and deploys.

To auth inside a volume-mounted container (so tokens land in the volume, not on the host):

```bash
# Option A: exec into running container
docker-compose exec ai-toolbox npx -y mcp-remote@0.1.38 https://mcp.groww.in/mcp 52155

# Option B: temporary container with same volume
docker run --rm -it --network host -v app-server_mcp_auth:/root/.mcp-auth node:20-alpine \
  npx -y mcp-remote@0.1.38 https://mcp.groww.in/mcp 52155
```

## Google Cloud OAuth setup for Gmail

Needed when n8n (or any app) sends email via Gmail API. Free, no credit card.

**Steps:**

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → create a project
2. **APIs & Services** → **Library** → search **Gmail API** → **Enable**
3. **APIs & Services** → **OAuth consent screen** → fill app name, support email, developer email → **Save**
4. The app starts in **Testing** mode — only users you add as "test users" can authorize. No Google verification needed for personal use.
5. **OAuth consent screen** → **Audience** → **Add users** → add your Gmail address
6. **APIs & Services** → **Credentials** → **Create Credentials** → **OAuth Client ID**
7. Application type: **Web application**
8. Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback` (for n8n)
9. Copy **Client ID** and **Client Secret** into n8n's Gmail credential page

| Field | Value |
|-------|-------|
| Testing mode | Only added test users can authorize; no review process |
| Production mode | Anyone can authorize; requires Google verification (days/weeks) |
| Redirect URI | Must match exactly what the consuming app expects |
| Cost | Free — OAuth credentials and Gmail API have no charge |
| Projects limit | 25 per free Google account |

## n8n API

n8n exposes a REST API for managing workflows programmatically.

**Authentication:** Generate an API key in n8n UI → **Settings** → **API** → **Create API Key**. Pass it as a header:

```bash
curl -s http://localhost:5678/api/v1/workflows \
  -H "X-N8N-API-KEY: <your-key>"
```

**Common operations:**

```bash
# List workflows
curl -s http://localhost:5678/api/v1/workflows -H "X-N8N-API-KEY: $KEY"

# Create a workflow
curl -s -X POST http://localhost:5678/api/v1/workflows \
  -H "X-N8N-API-KEY: $KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "My Workflow", "nodes": [...], "connections": {...}, "settings": {}}'

# Update a workflow (strip read-only fields first)
curl -s -X PUT http://localhost:5678/api/v1/workflows/<id> \
  -H "X-N8N-API-KEY: $KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "...", "nodes": [...], "connections": {...}, "settings": {}}'
```

**Read-only fields** that must be removed from PUT requests: `id`, `active`, `createdAt`, `updatedAt`, `versionId`, `activeVersionId`, `isArchived`, `triggerCount`. Sending them returns `400 request/body/<field> is read-only`.

## Deployment Issues — Real-World Examples

Problems we hit deploying to an Oracle Cloud VM, and what each one teaches.

### SPA (Single Page Application) behind a reverse proxy

An SPA is a web app where the server returns **one HTML file** and JavaScript renders the entire page. Examples: React, Vue, Angular, n8n.

The HTML references assets that the browser fetches as **separate requests**:

```html
<script src="/assets/app.js"></script>
<link href="/assets/style.css">
```

**The problem with a path prefix:**

If you serve an SPA at `/workflow/` behind a reverse proxy:

```
1. Browser: GET /workflow/           → Caddy forwards to n8n → returns HTML ✅
2. HTML says: <script src="/assets/app.js">
3. Browser: GET /assets/app.js       → goes to ROOT, not /workflow/ → Caddy catch-all → 404 ❌
```

The initial page loads, but every asset request goes to the **root path** because the HTML uses absolute paths (`/assets/...`). The reverse proxy's `strip_prefix` only handled the first request — the browser makes the asset requests independently.

**Why strip_prefix doesn't fix it:**

`strip_prefix` processes requests **arriving at Caddy**. But the browser doesn't add the prefix to asset URLs — it reads `/assets/app.js` from the HTML and requests exactly that. Caddy never sees `/workflow/assets/app.js`, so there's nothing to strip.

```
What you expect:
  Browser → /workflow/assets/app.js → Caddy strips /workflow → /assets/app.js → n8n ✅

What actually happens:
  Browser → /assets/app.js → Caddy catch-all → "app-server" text → blank page ❌
```

**Fix options:**

| Option | How | When to use |
|--------|-----|-------------|
| App knows its prefix | React: `homepage` in package.json. Next.js: `basePath`. Vue: `base` in router. App rewrites all asset URLs **and** serves assets at the prefixed path. | App properly supports base paths |
| Separate port/subdomain | Serve the SPA on its own port (`:8080`) or subdomain (`workflow.domain.com`). No prefix, no mismatch. | App doesn't support base paths (n8n's `N8N_PATH` is broken) |

**The rule:**

| App type | Path prefix (`strip_prefix`) | Separate port/subdomain |
|----------|------------------------------|------------------------|
| API (JSON only) | Works — no assets to load | Works |
| SPA (HTML + JS + CSS) | Only if the app fully supports base paths | Always works |

### Secure cookies over HTTP

Browsers refuse to set cookies marked `Secure` when the connection is plain HTTP (not HTTPS). The cookie is silently dropped — no error in the network tab, just a broken login.

**What happened:** n8n marks its session cookie as `Secure` by default. Accessing `http://140.238.229.137:8080` (no HTTPS) → browser drops the cookie → login fails → "secure cookie" error page.

**Fixes:**

| Fix | How | When |
|-----|-----|------|
| Add HTTPS | Get a domain + let Caddy auto-generate TLS certs | Production |
| `N8N_SECURE_COOKIE=false` | Env var tells n8n to issue non-Secure cookies | Development / testing over HTTP |

```yaml
# docker-compose.yml
environment:
  - N8N_SECURE_COOKIE=false  # temporary — remove when HTTPS is added
```

### CDN caching in CI/CD

`raw.githubusercontent.com` caches files for approximately **5 minutes**. If your deploy script downloads config files from this URL, it may get the **old** version immediately after a push.

**The trap:**

```
1. You push a change (e.g. add N8N_SECURE_COOKIE=false)
2. GitHub Actions triggers deploy within seconds
3. Deploy script: curl https://raw.githubusercontent.com/.../docker-compose.yml
4. CDN returns the CACHED (old) version — your change isn't in it
5. VM runs stale config, env var missing, login still broken
6. You check the repo — file looks correct — nothing makes sense
```

**Attempted fixes:**

- `?t=$(date +%s)` cache-busting query param — unreliable, CDN sometimes ignores it
- Manually SSH in and write files — works but defeats automation

**Proper fix:** SCP files directly from the CI runner to the VM. The runner already has your repo checked out via `actions/checkout` — no CDN involved.

```
Before (fragile):
  push → GitHub CDN (cached ~5min) → curl on VM → may get stale file

After (reliable):
  push → checkout on CI runner → SCP to VM → always latest
```

### Docker creates directory for missing mount source

When you bind-mount a **file** that doesn't exist on the host, Docker creates a **directory** with that name instead of failing.

```yaml
volumes:
  - ./Caddyfile:/etc/caddy/Caddyfile  # if Caddyfile doesn't exist on host...
```

**The cascade:**

```
1. docker-compose up — Caddyfile doesn't exist on VM
2. Docker creates DIRECTORY ~/apps/server-config/Caddyfile/
3. Caddy tries to read config from a directory → fails
4. Next deploy: curl tries to download Caddyfile → can't overwrite a directory with a file → silent failure
5. Caddyfile stays a directory, Caddy keeps failing
```

**Prevention:** ensure mount source files exist on the host **before** running `docker-compose up`. In CI, the file-copy step (SCP or curl) must happen before `docker-compose up -d`.

### SCP in GitHub Actions

`appleboy/scp-action` copies files from the CI runner directly to the server over SSH. The runner already has your repo checked out, so files are always the latest version — no CDN.

```yaml
steps:
  - uses: actions/checkout@v4  # repo files now on the runner

  # Step 1: copy config files to VM
  - uses: appleboy/scp-action@v0.1.7
    with:
      host: ${{ secrets.VM_HOST }}
      username: ${{ secrets.VM_USER }}
      key: ${{ secrets.SSH_PRIVATE_KEY }}
      source: "docker-compose.yml,Caddyfile"
      target: ~/apps/server-config

  # Step 2: restart services
  - uses: appleboy/ssh-action@v1
    with:
      host: ${{ secrets.VM_HOST }}
      username: ${{ secrets.VM_USER }}
      key: ${{ secrets.SSH_PRIVATE_KEY }}
      script: |
        cd ~/apps/server-config
        docker-compose down --remove-orphans || true
        docker-compose pull
        docker-compose up -d
```

Two-step pattern: SCP the files, then SSH to restart. Files always match what's in the repo at the commit that triggered the workflow.
