# Temporal Concepts

Reusable concepts for TypeScript services using Temporal. Structured as a learning path — each section builds on the previous.

---

## Part 1: Foundations

### What is Temporal

Temporal is a durable execution system. It records a Workflow's event history in the Temporal cluster, so a Worker can replay the history and continue after a process or pod fails. The application code does not manage retries, state persistence, or crash recovery — Temporal handles all of that.

### Core primitives

| Primitive | Role |
| --- | --- |
| **Workflow** | Coordinates durable business steps. Must be deterministic because Temporal replays its code from history. |
| **Activity** | Performs non-deterministic work such as database access, HTTP calls, or file I/O. |
| **Worker** | Polls a Task Queue, executes Workflow and Activity Tasks, and reports results. |
| **Client** | Starts, signals, queries, cancels, and schedules Workflows. |
| **Schedule** | Durable server-side timing configuration that starts Workflows; it is not an in-process cron timer. |

### How they communicate

```text
Client ── gRPC ──> Temporal Frontend / cluster
                        │ stores history and queues Tasks
Worker ── gRPC ──>      │
```

The Client and Worker do not call each other directly. The Client submits commands to Temporal and the Worker long-polls a Task Queue for work. All communication flows through the Temporal cluster.

### gRPC and the Temporal Frontend

gRPC is a typed binary RPC protocol that runs over HTTP/2. The Temporal SDK uses it internally; application code normally calls SDK methods rather than building gRPC messages itself.

The **Temporal Frontend** is the cluster API service exposed to SDKs, commonly on port `7233`. "Frontend" here means the Temporal API boundary, not a web user interface. The Temporal Web UI is a separate browser application and is not the SDK's gRPC endpoint.

Typical flow:

```text
Client  ── StartWorkflowExecution ──> Temporal
Worker  ── PollWorkflowTaskQueue ───> Temporal
Worker  <─ Workflow Task + history ── Temporal
Worker  ── completion/failure ───────> Temporal
```

---

## Part 2: Writing Workflows

### Workflow determinism and replay

A Workflow runs in a restricted V8 isolate. When it resumes, Temporal replays recorded events to rebuild its state without repeating completed Activities.

Keep direct I/O out of Workflow code: do not call a database, `fetch`, Node filesystem APIs, `Date.now()`, or `Math.random()`. Use Activities for I/O and Temporal's Workflow APIs for time, randomness, signals, queries, and cancellation.

```ts
import { proxyActivities } from '@temporalio/workflow';
import type * as activities from '../activities';

const { fetchData } = proxyActivities<typeof activities>({
  startToCloseTimeout: '5m',
});

export async function exampleWorkflow(): Promise<void> {
  await fetchData();
}
```

Import Activity types into Workflows, not Activity implementations. Type-only imports disappear at build time and keep Node-specific code out of the Workflow bundle.

### Activities: I/O, timeouts, retries, and heartbeats

Activities run in normal Node.js, so they can use databases, network clients, logs, and environment configuration. They must be safe to retry because Activity execution is at least once.

**Timeouts** control different stages of an Activity's life:

| Timeout | What it limits |
| --- | --- |
| `startToCloseTimeout` | One started Activity attempt. |
| `scheduleToStartTimeout` | How long work waits for a Worker before it is considered stuck. |
| `scheduleToCloseTimeout` | The complete Activity lifetime, including all retries. |
| `heartbeatTimeout` | Requires a long-running Activity to periodically prove it is still progressing. |

**Retries** are configured in the Workflow's `proxyActivities` options. `maximumAttempts: 3` means the original attempt plus at most two retries. `initialInterval` and `backoffCoefficient` control the wait between retries.

**Heartbeats** are for long-running Activities that can expose useful progress or a resumable checkpoint. Call `Context.current().heartbeat(details)` from the Activity; the retry receives the last details in `Context.current().info.heartbeatDetails`. Short, bounded Activities do not need heartbeats.

### Workflow vs Activity

They have complementary roles:

| | Workflow | Activity |
| --- | --- | --- |
| Purpose | Orchestrate steps, make decisions | Perform real work (I/O, DB, HTTP) |
| Environment | Deterministic V8 isolate (no I/O) | Normal Node.js (full access) |
| Durability | Replayed from history on crash | Retried by Temporal on failure |
| Contains | Calls to `proxyActivities`, timers, signals | Business logic, service calls |

A Workflow with a single Activity call may look like "just a wrapper", but the wrapper is the point: it gives the Activity Temporal's durable execution guarantees (retry, timeout, persistence, schedule-ability). Without the Workflow shell, the Activity is just a plain function with no crash recovery.

For multi-step business processes, the Workflow orchestrates several Activities in sequence or parallel, handling failures, compensations, and branching — things a plain function cannot survive across process restarts.

### proxyActivities: the bridge

#### Why it exists

Workflows run in a sandboxed V8 isolate with no access to Node.js APIs — they cannot import real functions that touch databases, HTTP, or the filesystem. But a Workflow still needs to call Activities. `proxyActivities` solves this: it creates fake (proxy) versions of Activity functions that the Workflow can call. When the Workflow calls a proxied function, it does not run the real code — it sends a command to Temporal saying "schedule this Activity by name". Temporal then dispatches the real function to a Worker running in normal Node.js. This indirection is what makes Activities retriable, timeboxed, and crash-recoverable.

#### How it works

`proxyActivities` is the only public API for calling Activities from Workflows in the TypeScript SDK. It creates a proxy object whose methods do not execute the real Activity code — they tell Temporal to schedule an Activity Task by name.

#### End-to-end flow

1. **Define the Activity** — a normal exported async function in `activities/`:
   ```ts
   export async function markStaleCeResearchesAsFailed(): Promise<number> {
     return ceResearchService.markStaleResearchesAsFailed(2);
   }
   ```

2. **Re-export from the barrel** — the Worker needs a map of Activity name → function:
   ```ts
   // activities/index.ts
   export { markStaleCeResearchesAsFailed } from './ceResearchCleanup.activities';
   ```

3. **Register with the Worker** — `Worker.create({ activities })` creates the lookup map `{ "markStaleCeResearchesAsFailed": fn }`.

4. **Proxy in the Workflow** — `import type` brings only the TypeScript shape, not Node.js code:
   ```ts
   import type * as activities from '../activities';
   const { markStaleCeResearchesAsFailed } = proxyActivities<typeof activities>({
     startToCloseTimeout: '5m',
   });
   ```
   The generic parameter `typeof activities` gives the proxy compile-time type safety. `import type *` is necessary because `proxyActivities` needs an object type to destructure — importing individual function types does not provide the object shape. The `type` keyword ensures no runtime code is pulled into the Workflow bundle.

5. **Runtime execution:**
   ```text
   Workflow calls markStaleCeResearchesAsFailed()
       ↓
   proxyActivities intercepts → ScheduleActivityTask command to Temporal
       ↓
   Temporal queues an Activity Task on the Task Queue
       ↓
   Worker polls, finds "markStaleCeResearchesAsFailed" in its activities map
       ↓
   Runs the real function (DB call happens here, in normal Node.js)
       ↓
   Returns result → Temporal records in history → Workflow receives the result
   ```

The Workflow never executes the Activity directly. It issues a command by name; Temporal dispatches it to a Worker that has the real implementation registered. This indirection is what makes the Activity retriable — Temporal owns the execution lifecycle, so if the Worker crashes or the Activity throws, Temporal can re-dispatch the task to any available Worker polling the same Task Queue.

---

## Part 3: Running Temporal

### Connections: Connection vs NativeConnection

The Client uses `Connection`, a TypeScript gRPC transport. The Worker uses `NativeConnection`, which connects Temporal's native Core bridge to the cluster. They are separate connection types and must not be shared.

**NativeConnection lifecycle:** When creating a Worker, connect `NativeConnection` and hold it in a local variable through `Worker.create()`. Assign it to module-level scope only after Worker creation and `run()` setup succeed. If creation fails, close the local connection. This prevents a concurrent shutdown from nulling a reference that creation still needs.

`@temporalio/worker` includes the native Core bridge. The bridge binary is built for glibc — Alpine's musl runtime is not supported (see Platform Requirements).

Keep the Temporal SDK packages at compatible versions. They cooperate across the client, Worker, Workflow sandbox, and Activity runtime.

### Worker: creation, run loop, and concurrency

**Creation:** `Worker.create()` takes the Task Queue name, registered Workflows and Activities, concurrency limits, and shutdown configuration. The Worker does not serve HTTP — it long-polls the Temporal cluster for tasks.

**`worker.run()` and the poll loop:** `worker.run()` starts the Worker's infinite poll loop — it continuously polls Temporal for tasks and executes them. It returns a Promise that **never resolves during normal operation** because the loop never ends on its own. It only resolves when `worker.shutdown()` is called and draining completes, or when an unrecoverable error occurs. Store this promise so the shutdown sequence can `await` it to know when draining is done.

**Concurrency limits:** `maxConcurrentWorkflowTaskExecutions` limits simultaneously processed Workflow Tasks on one Worker. `maxConcurrentActivityTaskExecutions` does the same for Activity Tasks. Extra Tasks wait in Temporal until capacity is available or another Worker polling the same queue receives them. These limits are per-Worker, not global — and they are not a limit on the total number of open Workflows, since waiting Workflows do not continuously use a Workflow Task execution slot.

### Workflow bundling

Workflow code must be bundled for the isolated Workflow runtime. The bundle is not a second copy of the application — it is the Workflow-only JavaScript payload loaded into the Worker sandbox. Activities remain normal Node.js modules.

Two approaches:

| Approach | When to use | How |
| --- | --- | --- |
| `workflowsPath` | Local development | Tells `Worker.create` where the Workflow module is; the SDK bundles it while the Worker starts. Convenient for fast iteration. |
| `workflowBundle.codePath` | Production | Points to an already-created bundle. Build it with `bundleWorkflowCode` during the Docker/CI build. Makes startup predictable and avoids runtime bundling work. |

Pre-bundling is explicit because the Temporal Worker needs deterministic Workflow code isolated from ordinary Node.js modules. Producing it at image-build time makes startup failures deterministic.

### Task Queues and multi-worker distribution

A Task Queue is a logical queue in Temporal, not a Kubernetes Service or a network address. The Workflow starter/Schedule and Worker must use the same Task Queue name.

**Multiple Workers on the same Task Queue:** If an application runs across N pods, each with its own Worker, all N Workers poll the same Task Queue. Temporal distributes each Task to exactly one eligible Worker. No duplicate execution occurs. If one Worker is at its concurrency limit, the Task goes to another with capacity — or waits in the queue.

This means scaling Workers horizontally (more pods) directly increases throughput, while the per-Worker concurrency limits prevent any single process from being overloaded.

---

## Part 4: Scheduling and Namespaces

### Schedules

Schedules are stored and evaluated by Temporal. They are durable server-side triggers (like cron), not in-process timers. A Schedule creates Workflow executions according to its timing configuration.

**Overlap policies** determine what happens when a new tick occurs before the prior Workflow finishes:

| Policy | Behavior |
| --- | --- |
| `SKIP` | Drops the new tick. Safe default for most periodic jobs. |
| `BUFFER_ONE` | Retains one pending tick; starts it after the current run finishes. |
| `ALLOW_ALL` | Starts every tick, allowing overlapping executions. |

**Catchup window** (`catchupWindow`): bounds how long a missed trigger may be started after the cluster becomes available again. Set it explicitly — without it, a cluster outage followed by recovery could fire many backlogged ticks at once.

**Reconciliation at startup:** A service normally reconciles its declared schedules at startup: create the Schedule, and if it already exists, update it to the declared definition. The SDK Schedule Client supports create-or-update behavior. Maintain a simple array of Schedule-definition builders, build definitions synchronously, and reconcile independent schedules concurrently with `Promise.all`. Do not swallow update errors.

### Namespaces and environment isolation

A namespace is Temporal's logical tenancy and retention boundary. A Task Queue lives within a namespace. Namespace creation is an administrative operation.

**Environment isolation:** If multiple environments (e.g., staging, dev instances) share one namespace, scope Task Queue names and stable identifiers (such as Schedule and Workflow IDs) by an environment identifier. Without this, a Worker in one environment can poll work created by another. Each environment's identifiers must be unique within the namespace.

**Namespace registration:** If an application is allowed to create a namespace at startup (e.g., for ephemeral/dev environments), handle these cases:

- **Already exists** — treat as success (the namespace was created by a prior startup or another instance).
- **Auth/permission/network failures** — propagate to the caller; do not swallow.
- **Never block** the main application startup indefinitely waiting for namespace creation.

### gRPC error handling for namespace operations

Temporal's raw protobuf types (like `registerNamespace`) require `Long` for duration fields (e.g., retention period in seconds). Use the `long` package: `Long.fromNumber(604800)` for seven days.

The SDK exports `isGrpcServiceError` to narrow gRPC failures. gRPC uses numeric status codes:

| Code | Name | Meaning |
| --- | --- | --- |
| 0 | OK | Success |
| 5 | NOT_FOUND | Resource does not exist |
| 6 | ALREADY_EXISTS | Resource already exists (e.g., namespace, Workflow ID) |
| 7 | PERMISSION_DENIED | Caller lacks permission |
| 14 | UNAVAILABLE | Transient failure, safe to retry |
| 16 | UNAUTHENTICATED | Invalid or missing credentials |

Check for specific codes rather than catching all gRPC errors uniformly. For example, when registering a namespace, only `ALREADY_EXISTS` (code 6) should be silently accepted — every other failure should propagate.

---

## Part 5: Operations

### Graceful shutdown and draining

On process termination, stop accepting or scheduling new work, ask the Worker to drain within `shutdownGraceTime`, wait for its run loop, then close the Worker connection and Client connection. Close dependencies used by Activities (database, cache) only after the Worker has stopped.

**Draining** is the Worker's controlled wind-down sequence:

1. **Stop polling** — tells Temporal "do not send me new tasks".
2. **Finish in-flight work** — currently running Workflow Tasks and Activities are allowed to complete, up to `shutdownGraceTime`.
3. **Resolve** — once all in-flight work finishes (or the grace time expires), `worker.run()` resolves and the process can safely close gRPC connections and downstream dependencies.

Without draining, a hard kill would interrupt an Activity mid-execution (for example, mid-database-write). Temporal would eventually retry the Activity on another Worker, but work is wasted and partial state may be left behind.

`shutdownGraceTime` is a drain window, not a guarantee that an arbitrarily long Activity finishes. Long Activities need appropriate timeouts, idempotency, and, where useful, heartbeat checkpoints so they can be retried safely.

**Container termination budget:** `shutdownGraceTime` must be ≤ the container's stop timeout (`docker stop -t` or Kubernetes `terminationGracePeriodSeconds`). If the drain window exceeds the container's kill timeout, Docker/Kubernetes sends SIGKILL before draining finishes — defeating graceful shutdown entirely. Budget: `terminationGracePeriodSeconds` ≥ drain time + post-drain cleanup (closing connections, flushing logs).

**Shutdown order matters:**

```text
1. Cancel pending supervisor retries
2. worker.shutdown() → drain in-flight tasks
3. await worker.run() promise → confirms drain is done
4. Close NativeConnection (Worker's gRPC)
5. Close Client Connection (Client's gRPC)
6. Close database / cache / downstream dependencies
```

Closing dependencies before the Worker finishes would break in-flight Activities that use them.

### Runtime supervision vs SDK reconnect

The SDK and the application supervisor handle different failure domains:

| Concern | Who handles it |
| --- | --- |
| Transient gRPC disconnects while a healthy Worker is running | SDK's built-in reconnect (automatic, transparent) |
| Startup failure (can't connect, can't create Worker) | Application supervisor |
| `worker.run()` exits unexpectedly | Application supervisor |
| Namespace registration failure | Application supervisor |

**Supervisor pattern:** Track runtime states — `connecting`, `running`, `degraded`, `stopping`. On failure:

- Clean up partially created Client/Worker connections.
- Record the failure in logs, health state, and metrics.
- Retry with capped exponential backoff: 1, 2, 4, 8, 16, 30 seconds, then ~30 seconds with jitter.
- Add jitter to avoid all instances reconnecting simultaneously.
- Continue retrying until initialization succeeds or application shutdown begins.

The supervisor must not:
- Make the HTTP server unavailable because Temporal is temporarily unreachable.
- Call `process.exit()` on Temporal failure.
- Block application startup indefinitely.

### Platform requirements

`@temporalio/worker` loads a native Core bridge built for **glibc**. Runtimes using **musl** (Alpine Linux) are not supported by the standard bridge binary. Use a glibc-based image such as `node:20-slim` (Debian).

This affects Docker builds: if the application previously used Alpine, both the builder and runtime stages must switch to a glibc-based image when adding Temporal Worker support.

### Local development

The Temporal CLI development server provides a local Temporal service and Web UI. A command such as the following stores its local state in SQLite:

```bash
mkdir -p .temporal
temporal server start-dev --namespace <your-namespace> --db-filename .temporal/temporal.db
```

- The `--namespace` flag pre-creates the namespace (the dev server otherwise only creates `default`).
- The SQLite file keeps local Workflow history across restarts.
- The SDK connects to the dev server's gRPC address (default `localhost:7233`); the Web UI URL (default `localhost:8233`) is for humans, not SDK connections.
- Keep local state directories (`.temporal/`) out of version control.
- Local development should use `workflowsPath` for fast iteration (no separate bundle-build step after every source change).
