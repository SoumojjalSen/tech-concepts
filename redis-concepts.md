# Redis / ElastiCache — Supply Mission Control

Reference doc for the ElastiCache Redis provisioned for SMC (Terraform PR #1238, already merged).

---

## What was provisioned

| Config | Test | Prod |
|--------|------|------|
| Cluster name | `test-redis-supply-mission-control` | `supply-mission-control-prod` |
| Node type | `cache.t4g.micro` (0.5 GB RAM) | same |
| Redis version | 7.1 | same |
| Mode | single node, no replica | same |
| TLS in-transit | off | off |
| At-rest encryption | on | on |
| AZ | `ap-south-1a` (Mumbai) | `us-east-1a` (US East) |
| Internal DNS (primary) | `test-redis-supply-mission-control.internal.test-headout.com` | `supply-mission-control-prod-redis.internal.headout.com` |
| Internal DNS (read-only) | `test-redis-supply-mission-control-ro.internal.test-headout.com` | none — production is single-node |
| Port | 6379 | 6379 |

---

## Design decisions — rationale

**`cache.t4g.micro` (0.5 GB)** — sufficient for the measured filter snapshot. The real top-1000 sample used about 2.32 MiB for 1,000 Tour Groups, 6,061 Tours, 1,979 Vendors, and 19,659 exact relationships. Even three overlapping snapshot versions are roughly 7 MiB before normal Redis overhead, leaving substantial headroom. Monitor memory and evictions rather than treating the estimate as a capacity guarantee.

**Single node, no replica** — appropriate for the current phase. Redis has no business-critical consumer yet, and SMC starts in fail-open mode when Redis is unavailable. There is no automatic request-time BigQuery fallback. Revisit a replica and Multi-AZ when filter availability receives an explicit service-level requirement.

**TLS in-transit off** — fine. Traffic stays within the VPC on private subnets. All other SMC-adjacent services use the same pattern. No TLS = simpler client config, no cert management.

**At-rest encryption on** — correct.

---

## DNS naming asymmetry (heads up)

The hostnames follow the org's existing convention — test and prod are NOT symmetric:

- Test: `test-redis-{name}.internal.test-headout.com` (prefix pattern)
- Prod: `{name}-prod-redis.internal.headout.com` (suffix pattern)

Different `REDIS_HOST` env vars are needed per environment. Cannot use the same value template.

---

## SMC application wiring

The generic ioredis integration uses one lazy, long-lived client per SMC process. It connects without TLS because the provisioned caches have in-transit encryption disabled.

| Environment | Host |
|---|---|
| Local | `localhost` |
| ODE | `${DEPLOY_NAMESPACE}-redis-master.dex.svc.cluster.local` |
| Test | `test-redis-supply-mission-control.internal.test-headout.com` |
| Production | `supply-mission-control-prod-redis.internal.headout.com` |

Port `6379` is used in every environment. Local Redis runs through `yarn redis:local`; `yarn redis:cli` opens the CLI in that container.

ODE uses Kubernetes service DNS: `<service>.<namespace>.svc.cluster.local`. `${DEPLOY_NAMESPACE}` is the ODE deployment namespace, `-redis-master` completes that ODE's Redis service name, `dex` is the namespace where Redis runs, and `svc.cluster.local` is the cluster service-DNS suffix. The full name is required because SMC runs outside `dex`; `6379` is Redis's service port.

The client configuration is:

- `lazyConnect: true`
- `enableOfflineQueue: false`
- `enableReadyCheck: true`
- `connectTimeout: 5_000`
- `commandTimeout: 5_000`
- `socketTimeout: 10_000`
- `maxRetriesPerRequest: 5`
- `autoResendUnfulfilledCommands: false`
- The ioredis v6 default reconnect strategy

---

## Redis concepts

### What Redis is

Redis is an in-memory data store. It keeps data primarily in RAM, so reads and writes are usually much faster than disk-backed database operations. Redis can be used as a cache, a temporary state store, a queue, a distributed coordination mechanism, or a primary database when its data model and durability options fit.

Redis is not automatically a source of truth. When it is used as a cache, another system normally owns the authoritative data and can rebuild Redis after data expires or is lost.

### Redis OSS and ElastiCache

Redis OSS is the Redis server and protocol. Amazon ElastiCache is AWS infrastructure that operates a compatible Redis engine and manages networking, instance replacement, monitoring, backups, and maintenance. An application still uses a normal Redis client and normal Redis commands.

### Server, client, connection, and command

- The Redis server owns the data and executes commands.
- A Redis client library translates application method calls into the Redis wire protocol.
- A client object manages a TCP connection, command queues, timeouts, and reconnection.
- A command is an operation such as `GET`, `SET`, `HGET`, or `EXPIRE`.

A long-lived client is created once per application process and reused. It does not mean that one physical socket can never change: after a network failure, the same client object can replace its socket by reconnecting. Reusing the client avoids a TCP handshake and a new connection for every command.

A **socket** is an operating-system object that a program uses to send and receive network data. A connected TCP socket tracks the local and remote IP addresses and ports, the TCP connection state, and incoming and outgoing byte buffers. Here, the application socket and the Redis server socket are the two endpoints of one TCP connection. The client writes Redis commands to its socket, TCP transports the bytes, and Redis sends replies back over the same connection. The socket is one endpoint; the TCP connection is the communication link between both endpoints.

After TCP connects, the client may still need to complete Redis-level setup before ordinary commands are safe to use. Depending on the deployment, this can include TLS negotiation, authentication, selecting a database, setting the client name, and checking that Redis has finished loading. “Redis handshake” is convenient shorthand for this initialization; Redis does not define one single command formally called a handshake.

The ioredis `connectionName` option sends Redis's `CLIENT SETNAME` command for that connection. Redis exposes the name through `CLIENT LIST`, which helps operators identify which application or client role owns each connection. The name is diagnostic metadata: it does not select a database, create a separate connection pool, or affect keys and commands. Names do not have to be unique because every application pod can create its own connection with the same application name.

Normal Redis commands can share one connection because replies are matched to commands in order. Pub/Sub and blocking commands need dedicated connections because they change how that connection behaves.

### DNS

DNS translates a hostname into an IP address. Applications should use the cache hostname rather than storing an IP address because infrastructure can replace a Redis node and point the same hostname at a new endpoint.

DNS does not retry Redis commands or move application data. It only helps the client discover the current network address. A reconnect causes the client and operating system to resolve the hostname again according to their DNS caching behaviour.

### Primary, replica, Multi-AZ, and automatic failover

- A **primary** accepts writes and serves reads.
- A **replica** asynchronously copies the primary's data and can serve reads or be promoted after a primary failure.
- **Multi-AZ** places the primary and at least one replica in different Availability Zones so one zone failing does not remove every copy.
- **Automatic failover** promotes a healthy replica and updates the managed endpoint when the primary fails.

Multi-AZ therefore requires a replica; it is not a switch that makes a single node highly available. It improves availability but costs another node and does not prevent application errors during the failover interval. A derived, non-critical cache can begin with one node and add Multi-AZ when its availability requirement justifies the cost.

### Cluster mode and sharding

With cluster mode disabled, one primary owns the complete keyspace. Replicas, if configured, copy that same keyspace. This is the simplest topology and fits data that remains within one node's memory and throughput.

With cluster mode enabled, Redis partitions keys across multiple primary shards. It increases capacity and throughput, but adds topology, cross-slot command restrictions, resharding, and more nodes. It is useful when one primary is a measured bottleneck, not merely because Redis is used in production.

### At-rest and in-transit encryption

- **At-rest encryption** protects data stored on disks, snapshots, and backups. It does not encrypt bytes travelling over the network.
- **In-transit encryption** uses TLS between the application and Redis. It protects traffic from network interception but requires TLS-capable endpoints and client configuration.

Private VPC routing limits who can reach the endpoint, but it is not encryption. These controls solve different threats and should be chosen from the data classification and infrastructure policy.

### Connection lifecycle

Typical client states are:

1. `wait`: lazy connection is enabled and no connection has started.
2. `connecting`: the client is opening a TCP connection.
3. `connect`: the TCP connection is open.
4. `ready`: Redis is ready to accept ordinary commands.
5. `close`: the socket closed.
6. `reconnecting`: the client is waiting before another connection attempt.
7. `end`: the client stopped reconnecting or was deliberately closed.

`connect` and `ready` are different. A TCP socket can exist while Redis is still loading data and cannot serve ordinary commands.

### `QUIT` and `disconnect`

`QUIT` is a Redis protocol command sent to the server. It needs an open, writable connection, so it is appropriate when the client is `ready`: Redis can acknowledge the command and then close the connection cleanly.

In `wait`, no connection exists yet; in `connecting`, the connection or protocol handshake is incomplete; and in `close` or `reconnecting`, the usable socket has already been lost. A `QUIT` command therefore cannot be sent reliably in these states. With the offline queue disabled it fails immediately; queueing it until reconnection would make application shutdown wait for Redis to recover.

`disconnect(false)` is local: it immediately closes any current socket and tells ioredis not to reconnect. It does not need Redis to be reachable or acknowledge anything, so it is the shutdown fallback when graceful `QUIT` is unavailable or fails. The `false` argument means “do not reconnect.”

### Client configuration

#### `lazyConnect`

- `true`: constructing the client does not open a connection. Calling `connect()` or issuing the first command starts connection establishment.
- `false`: the client starts connecting as soon as it is constructed.

With the offline queue disabled, a first command can trigger connection establishment but still fail because Redis is not ready yet. A consumer that requires a warm connection should explicitly await `connect()` and handle failure before issuing commands.

#### `enableOfflineQueue`

- `true`: commands issued before Redis is ready are held in application memory and sent later.
- `false`: those commands fail immediately.

Queuing can hide a short outage, but callers may time out while their commands remain queued. A large outage can also build a large memory backlog and execute stale writes after recovery.

#### `enableReadyCheck`

- `true`: after opening the socket, the client checks whether Redis has finished loading data before declaring itself ready.
- `false`: the open socket is treated as ready immediately.

#### `connectTimeout`

This is the maximum time allowed for one connection attempt. It does not limit the lifetime of the client or the total time spent across later reconnect attempts.

#### `commandTimeout`

This is the maximum time a command promise may wait for a reply. The application receives a timeout error when the limit is reached. The server may still have executed a write before the response was lost.

#### `socketTimeout`

This is the maximum time an established socket may remain silent while a command is awaiting a response. When it expires, ioredis destroys the unresponsive socket so its reconnect strategy can establish a replacement connection.

#### `maxRetriesPerRequest`

This limits how many reconnect cycles an unfulfilled request can survive before the client rejects it. It is not a request to execute every Redis command that many times.

#### `retryStrategy`

The reconnect strategy decides whether and when the client opens another connection after disconnection. Backoff prevents every application pod from reconnecting continuously. Jitter gives pods slightly different delays and reduces a synchronized reconnect spike. ioredis v6 defaults to exponential backoff capped at five seconds with random jitter.

An explicit `connect()` starts and awaits the initial connection; it is not itself a reconnect loop. If that connection later closes unexpectedly, ioredis invokes its reconnect strategy automatically. Catching a rejected initial `connect()` does not disable those later attempts. Reconnection stops when the client is deliberately closed with `quit()` or `disconnect(false)`, or when the reconnect strategy returns no delay.

#### `autoResendUnfulfilledCommands`

- `true`: after reconnection, the client automatically resends commands that were written but did not receive replies.
- `false`: the client rejects or leaves those commands for the application to handle.

Automatic resend is dangerous for a non-idempotent write. Redis may have applied the first write even though its reply never reached the client.

### Why a write can execute twice

Consider an increment:

1. The application sends `INCR stock-counter`.
2. Redis increments the value.
3. The network drops before the reply reaches the application.
4. The application cannot know whether Redis executed the command.
5. Retrying can increment the value a second time.

For an idempotent command such as `SET snapshot-version complete-payload`, executing the same write twice normally produces the same final value. For counters, list pushes, payments, and state transitions, duplicate execution can change the result. Safe retry design therefore depends on the command and business operation, not only on the client setting.

### Core data types

#### String

A Redis string is a binary-safe value stored under one key. It can contain text, JSON, integers, or arbitrary bytes.

```text
SET user:42:name "Ada"
GET user:42:name
```

`GET` reads one key and is O(1). `MGET` reads several independent string keys in one network round trip and is O(N) for N requested keys.

```text
MGET user:42:name user:42:country user:42:language
```

#### Hash

A Redis hash stores multiple field/value pairs under one top-level key.

```text
HSET user:42 name "Ada" country "UK" language "English"
```

- `HGET user:42 name` reads one field and is O(1).
- `HMGET user:42 name country` reads named fields from the same hash and is O(N) for N fields.
- `HSCAN user:42 0 COUNT 100` incrementally visits fields without requesting the whole hash at once.

`HSCAN` returns a new cursor and a batch. Continue using the returned cursor until it becomes `0`. `COUNT` is a hint, not an exact page size. A full scan is O(N), is not ordered, and may return duplicates while the hash is changing, so consumers must tolerate and deduplicate them.

#### Other useful types

- Lists preserve insertion order and support pushing and popping values.
- Sets store unique unordered members.
- Sorted sets store unique members ordered by a numeric score.
- Streams store append-only entries with consumer-group support.

Choose a type because its native commands match the access pattern; do not encode every structure as one large JSON string by default.

### Expiration and TTL

A TTL is the remaining lifetime of a key. `EXPIRE key 60` makes the key eligible for deletion after 60 seconds. `SET key value EX 60` stores a value and its expiration atomically.

Expiration controls staleness and memory use. It is not an exact scheduling system: Redis guarantees that an expired key will no longer be returned, but physical memory cleanup can occur lazily or in background cycles.

Adding random variation to many identical TTLs can prevent thousands of keys from expiring and rebuilding at exactly the same moment.

### Serialization

Redis stores bytes, not JavaScript objects. An object must be encoded before writing and decoded after reading, commonly with JSON.

```text
object -> JSON.stringify -> Redis -> JSON.parse -> validated object
```

Parsing JSON only proves that the bytes contain valid JSON. Validate the decoded shape at the Redis boundary because old deployments, manual changes, or corrupt data can produce a value the current application does not understand.

JSON cannot represent every JavaScript type directly. Dates become strings, `BigInt` throws without custom handling, and class behaviour is lost.

### Round trips, MGET, HMGET, and pipelining

Network round trips often cost more than the Redis command itself.

- `MGET` batches reads of multiple string keys.
- `HMGET` batches reads of multiple fields in one hash.
- A pipeline sends multiple different commands together and receives their replies together.

Pipelining reduces network waits but is not atomic. Other clients can execute commands between commands in a pipeline.

### Transactions and atomicity

Redis executes each individual command atomically. `MULTI` and `EXEC` queue a group of commands and execute them without another client's commands interleaving.

A Redis transaction does not provide SQL-style rollback. If a queued command fails during execution, other commands can still execute. Lua scripts are useful when multiple reads and writes must be evaluated atomically on the server.

### Large keys and large values

A large value can cause:

- high memory usage;
- longer serialization and parsing in Node.js;
- longer network transfers;
- slower deletion or expiration work;
- expensive full reads when only a small part is needed;
- latency spikes that affect unrelated commands on Redis's command-processing thread.

Prefer access-pattern-sized values. Split data when consumers regularly need independent parts, but do not create thousands of keys that must all be fetched individually for one request. Measure serialized bytes, field counts, command latency, and memory before choosing a boundary.

Avoid `KEYS pattern` in application traffic because it scans the complete keyspace at once. Use `SCAN` for incremental operational iteration, while remembering that a full scan still performs O(N) total work.

### Cache failure behaviour

A cache consumer should decide explicitly:

- fail open: bypass Redis and use the source of truth;
- fail closed: return an unavailable response because no safe fallback exists;
- serve stale: use an older validated value;
- rebuild asynchronously: let background work restore the cache.

The Redis client cannot choose this business behaviour. Connection retries only restore transport; they do not decide whether old data is acceptable or whether a source query should run again.

SMC currently applies fail-open only to application startup: a failed Redis connection is logged and the process can still serve non-Redis features. A future endpoint whose only filter source is Redis cannot silently invent data; it should either serve a previously validated snapshot or return a retryable feature-level error.

### Distributed locks and workflow coordination

A Redis lock such as `SET lock-key token NX PX milliseconds` lets only one process hold a lease. Correct use also needs lease renewal for long work and token-checked release so one process cannot delete another process's newer lock.

A durable workflow scheduler can own the same coordination. Temporal's overlap policy can skip a new scheduled run while the previous run is active, so adding a second Redis refresh lock would duplicate responsibility. Retries remain safe when the workflow supplies a fixed source-data cutoff and rewrites the same unpublished snapshot version before moving the active-version pointer.

### Refresh cadence, snapshots, and TTL

A versioned snapshot is written under new keys and becomes visible only when one small active-version pointer changes. Readers therefore see either the old complete snapshot or the new complete snapshot, never a partially written mixture.

The future filter snapshot is expected to refresh twice weekly. Because the longest interval can be roughly four days, an eight-day TTL preserves the last good snapshot through one missed refresh. TTL is a cleanup and staleness boundary; it is not the refresh scheduler.

### Observability and security

Useful signals include connection state changes, command latency, timeouts, error rates, reconnect counts, memory usage, evictions, key counts, and cache hit ratio.

Never log Redis passwords, complete connection URLs, or cached user/business payloads. Prefer a stable client name and bounded metric labels. High-cardinality values such as keys, user IDs, and query text should not become metric labels.
