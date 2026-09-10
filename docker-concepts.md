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
