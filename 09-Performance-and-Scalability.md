# Microservices Study Notes — Section 9

## 9. Performance and Scalability

---

### 9.1 Scalability

---

#### 1. Horizontal Scaling

**Horizontal scaling (Scale Out)** increases capacity by running **more instances/pods** of a microservice across multiple nodes behind a load balancer:
- **Strengths:** Near-unlimited cloud elasticity, built-in fault tolerance (losing 1 of 10 pods leaves 90% capacity up), zero-downtime rolling updates.
- **Prerequisite:** Services must be **stateless** (externalize sessions, files, and locks to Redis/DB/S3).

---

#### 2. Vertical Scaling

**Vertical scaling (Scale Up)** increases capacity by upgrading a single node's hardware (more vCPUs, RAM, or faster NVMe IOPS):
- **Strengths:** Zero code changes, preserves strong local ACID consistency (critical for primary PostgreSQL/Oracle DBs).
- **Limitations:** Hits hard physical hardware ceilings, exponential cost curves at top tiers, single-node blast radius, and often requires a restart.

| Dimension | Horizontal Scaling (Scale Out) | Vertical Scaling (Scale Up) |
| --- | --- | --- |
| **Mechanism** | Add more service pods/nodes (`3 → 30` pods) | Bigger machine (`4 vCPU → 64 vCPU`) |
| **Limit** | Very high (limited by downstream DB/bottlenecks) | Hardware ceiling of 1 physical host |
| **Fault Tolerance** | **High** (redundancy across AZs) | **Low** (SPOF unless paired with failover replica) |
| **Best Fit** | Stateless microservices, Kafka consumers, Redis Cluster | Primary relational DBs, heavy single-node analytics |

---

#### 3. Stateless Services

A **stateless service** stores **zero client session state or durable data in local JVM memory or container disk** between HTTP requests:
- Any pod can serve any request interchangeably—you can kill, restart, or scale pods from 2 to 50 seamlessly.
- **Externalize State:** Store auth in signed **JWTs** or **Redis** sessions, shopping carts in **Redis/DB**, uploaded files in **S3/Object Storage**, and scheduled job locks in **ShedLock/DB**.

---

#### 4. Stateful Services

A **stateful service** maintains persistent local data or node identity across restarts (**PostgreSQL, MySQL, Kafka brokers, Elasticsearch, Redis Cluster**):
- In Kubernetes, deployed using **`StatefulSets`** (instead of `Deployments`), providing:
  - Stable, predictable network hostnames (`kafka-0`, `kafka-1`, `kafka-2`)
  - Dedicated persistent storage (`PersistentVolumeClaim` pinned to each ordinal pod)
  - Ordered startup, shutdown, and rolling upgrades
- **Scaling Challenge:** Cannot simply duplicate pods—scaling requires data replication, partition rebalancing, or **sharding**.

---

#### 5. Load Balancing

Distributing traffic evenly across horizontally scaled instances using **Round Robin**, **Least Connections**, or **Power of Two Random Choices (P2C)**:
- **Avoid Sticky Sessions (Session Affinity) in Microservices!** Pinning a user to Pod 3 prevents autoscaling from evenly distributing load onto newly booted Pod 4 and breaks traffic when Pod 3 scales down.
- **HTTP/2 & gRPC Note:** Because HTTP/2 multiplexes many streams on 1 TCP connection, L4 Kubernetes `ClusterIP` pins traffic to 1 pod; use L7 Envoy/Service Mesh or gRPC client-side load balancing as you scale pods!

---

#### 6. Autoscaling

Automatically adjusting compute capacity in response to live demand:
- **HPA (Horizontal Pod Autoscaler):** Scales Kubernetes pod replicas based on CPU/Memory OR custom metrics (RPS, P95 latency).
- **KEDA (Kubernetes Event-Driven Autoscaling):** Scales consumer pods directly on **Kafka Consumer Lag** or **RabbitMQ queue depth** (can even scale workers to `0` when idle!).
- **VPA (Vertical Pod Autoscaler) & Cluster Autoscaler / Karpenter:** Right-sizes pod CPU/RAM requests and provisions new cloud VM nodes when pods run out of cluster capacity.
- **The JVM Cold-Start Trap:** Traditional Spring Boot + JVM takes 15–45s to start and warm up JIT, meaning reactive HPA can lag sudden flash-sale bursts! Mitigate via **Quarkus / GraalVM Native Image (<100ms startup)**, **AppCDS / Spring Boot AOT**, **scheduled pre-scaling**, and keeping ~30% spare headroom.

---

#### 7. Scaling Bottlenecks

Adding 10x more microservice pods rarely gives 10x throughput because downstream shared bottlenecks saturate (**Amdahl's Law**):
1. **Shared Primary Database:** 50 pods × 20 Hikari connections = **1,000 connections** hammering 1 PostgreSQL primary (connection exhaustion + lock contention).
2. **Kafka Partition Ceiling:** Having **6 partitions** on `orders` topic means scaling consumer pods from 6 to **20** leaves **14 pods completely idle**!
3. **Single Hot Key / Row Lock:** Flash sale where 5,000 users buy the exact same `SKU-1` concurrently—all transactions serialize behind 1 PostgreSQL row lock!
4. **Central Synchronous Dependencies:** Uncached calls to an external rate-limited API or central Auth introspection endpoint.

---

#### 8. Database Scaling

Follow this **systematic database scaling ladder** before jumping to complex sharding:
1. **Indexes & Query Tuning:** Fix missing indexes, N+1 queries, and `SELECT *` (often gives 10x–100x gains free!).
2. **Caching (Caffeine + Redis):** Offload 80%–95% of repeated reads from the DB.
3. **Connection Pooling / Proxy (`PgBouncer` / `RDS Proxy`):** Multiplex thousands of app pod connections onto optimal small DB connections.
4. **Vertical Scaling:** Upgrade primary DB vCPU/RAM/NVMe IOPS.
5. **Read Replicas:** Offload read-heavy queries.
6. **Table Partitioning:** Split giant tables (e.g., `orders_2026_11`) on the same DB server for faster index scans & instant old-data drops.
7. **Database Sharding (Last Resort!):** Split data across independent DB servers.

---

#### 9. Read Replicas

Maintain a **Primary (Read/Write)** node streaming WAL/binlog asynchronously to **1 or more Read Replicas (Read-Only)**:
- Route reporting, dashboards, search, and Spring `@Transactional(readOnly = true)` queries to Read Replicas (`AbstractRoutingDataSource`).
- **Replication Lag Pitfall (`Read-Your-Own-Writes` Violation):**
  - User updates profile on Primary (`t=0`), refreshes page 20ms later, and hits a Read Replica lagging by `100ms` → user sees old data or `404`!
  - **Fix:** Route immediate post-write reads from that user briefly to Primary (or return updated DTO directly from the `PUT`/`POST` response!).

---

#### 10. Database Sharding

**Sharding** horizontally partitions rows across multiple independent database servers (`Shard 0`, `Shard 1`, `Shard 2`) using a **Shard Key** (`tenant_id`, `customer_id`, `order_id`):

| Sharding Strategy | How It Works | Pros & Cons |
| --- | --- | --- |
| **Hash / Consistent Hashing** | `shard = hash(shardKey) % N` (Consistent Hashing minimizes row moves when $N$ grows) | **Even write distribution**; destroys range queries across keys |
| **Range-Based** | IDs `1–10M` on Shard 1, `10M–20M` on Shard 2 (or by date/alphabet) | Fast range scans; **hotspot risk** if all new traffic hits latest shard |
| **Directory / Lookup-Based** | Central lookup table maps `tenantId → shardId` | Flexible per-tenant isolation/moves; lookup table can become SPOF unless cached |

- **Sharding Costs:** No cross-shard SQL `JOIN`s, no cross-shard ACID transactions, risk of **Hot Shards** (one giant enterprise customer consuming 80% of Shard 2), and painful operational resharding.

---

#### 11. Caching Strategies

Scale reads exponentially and protect databases by layering a **Multi-Tier Caching Pyramid**:

```text
[Client / Browser] ──► [CDN (Cloudflare/CloudFront)] ──► [API Gateway Cache] ──► [L1 Local (Caffeine)] ──► [L2 Distributed (Redis)] ──► [DB]
```

Each tier trades freshness for lower latency and higher throughput—always cache closer to the user for static/public data and use L1/L2 inside microservices for hot domain data.

---

### 9.2 Caching

---

#### 1. Local Caching

**Local (In-Process) Caching** stores entries directly inside the microservice's JVM heap (e.g., **Caffeine** using the near-optimal **W-TinyLFU** eviction algorithm):
- **Pros:** **Nanosecond–microsecond latency** (zero network hop, zero JSON deserialization overhead if storing objects directly!), immune to Redis network outages.
- **Cons:** Limited by each pod's JVM heap (`maximumSize` mandatory to prevent OOM!), cold on pod restart, and **inconsistent across pods** (Pod 1 updates DB; Pod 2 still holds stale local cache until TTL expires or an invalidation event arrives via Kafka/Redis PubSub!).

---

#### 2. Distributed Caching

**Distributed Caching** stores shared state in an external in-memory cluster (**Redis**, **Valkey**, **Memcached**, **Hazelcast**):
- **Pros:** **Single consistent cache view** across all 50 pods, survives app pod restarts/deployments, holds tens/hundreds of GBs independently of app heap.
- **Cons:** `~0.5–2ms` network RTT + serialization overhead; becomes a critical dependency if app relies on it blindly without DB fallback.
- **Two-Level Cache (`L1 Caffeine + L2 Redis`):** Keep ultra-hot small keys (feature flags, currency rates, product categories) in L1 Caffeine (`TTL = 30s`) backed by L2 Redis (`TTL = 10m`)—slashing Redis network traffic by 95%!

---

#### 3. Redis

**Redis** (and its open-source Linux Foundation fork **Valkey**) is the standard microservice distributed cache and in-memory data store:
- **Single-Threaded Event Loop (for command execution):** Executes commands atomically in memory (`O(1)`/`O(log N)`) with zero lock contention at 100k+ ops/sec per shard—**warning:** avoid blocking `O(N)` commands like `KEYS *` (use `SCAN`) or huge `HGETALL` on 1M-field hashes in production!
- **Essential Data Structures:**
  - `Strings` (cached JSON DTOs, `SET NX PX` locks)
  - `Hashes` (user session/cart attributes)
  - `Sorted Sets (ZSET)` (leaderboards, exact sliding-window rate limiters!)
  - `Sets` / `Lists` / `Streams` / `HyperLogLog` (approximate unique visitor counts in 12KB!).
- **Persistence & HA:**
  - `RDB` (point-in-time snapshots) vs `AOF` (append-only write log).
  - **Redis Sentinel** (HA primary-replica automatic failover for 1 dataset) vs **Redis Cluster** (horizontal sharding across **16,384 hash slots** + replicas).
- **Eviction Policy:** Set **`allkeys-lru`** or **`allkeys-lfu`** for pure caches (prevents `OOM command not allowed` write failures when memory fills!).

---

#### 4. Cache-Aside

Also called **Lazy Loading**—the most widely used caching pattern in Spring Boot / Quarkus (`@Cacheable`, `@CacheEvict`):

```text
READ:  App ──► Check Redis ──► [HIT]  Return immediately!
                           └──► [MISS] Query DB ──► Write to Redis (with TTL) ──► Return

WRITE: App ──► Update DB ──► DELETE key from Redis (invalidate)
```

- **Pros:** Caches only data actually requested; service still functions (slower) if Redis goes down.
- **Cons:** Cache miss penalty (3 hops on first read); brief race window for stale data under concurrent read+write.

---

#### 5. Read-Through

Application reads solely from the **Cache Provider**, which sits inline in front of the database:
- On a **miss**, the **cache library/service itself** knows how to load the missing row from the DB, store it, and return it to the application (application code stays clean of cache-miss wiring).

---

#### 6. Write-Through

Application writes to the **Cache**, and the cache **synchronously writes through to the DB** inside the same call before returning:
- **Pros:** Cache is always **fresh and consistent** with the database; never suffers cold read misses on newly written items.
- **Cons:** Higher write latency (2 synchronous writes); fills cache memory with newly written data that may never be read unless paired with short TTLs.

---

#### 7. Write-Behind

Also called **Write-Back**: Application writes to the **Cache** and gets an **instant success ack**, while the cache asynchronously batches and flushes dirty writes to the DB later:
- **Pros:** Blazingly fast write latency; coalesces 100 rapid updates to the same key (e.g., video view counters) into 1 batched DB `UPDATE`!
- **Cons:** **Risk of data loss** if the cache node crashes before flushing dirty writes! Only use for high-volume counters/telemetry, never financial ledgers.

| Pattern | Read Miss Behavior | Write Behavior | Consistency | Data Loss Risk |
| --- | --- | --- | --- | --- |
| **Cache-Aside** | App loads DB & fills cache | App writes DB, **deletes** cache | Eventual | None |
| **Read-Through** | Cache provider loads DB | Typically paired with Write-Through | Eventual / Strong | None |
| **Write-Through** | Loads DB on miss | App writes Cache → sync to DB | **Strong** | None |
| **Write-Behind** | Loads DB on miss | App writes Cache → **async** batch to DB | Eventual | **Yes (if cache crashes!)** |

---

#### 8. Cache Invalidation

##### Why Writes Should `DELETE` Instead of `UPDATE` the Cache!
Suppose Thread A and Thread B update the same product concurrently under **Cache-Aside**:
- If both **update** Redis (`SET product:1 ...`), Thread A writes DB (`price=$10`), Thread B writes DB (`price=$20`), Thread B updates Redis (`$20`), then slow Thread A overwrites Redis (`$10`) → **permanently stale cache until TTL!**
- By **deleting (`DEL`)** the cache key *after* DB commit (or publishing a **Debezium CDC / Outbox event** to invalidate Redis asynchronously), the next reader fetches the true `$20` from DB!

---

#### 9. Cache Expiration

Removing stale/unused entries cleanly:
- **Time-Based Expiration (TTL):** Redis checks expiry lazily on read AND samples expired keys periodically in the background.
- **Capacity Eviction Policies (When RAM Hits `maxmemory`):**
  - **`LRU` (Least Recently Used):** Evicts items untouched longest.
  - **`LFU` / `W-TinyLFU` (Least Frequently Used):** Protects hot popular items from being pushed out by a one-time batch scan!

---

#### 10. TTL (Time-To-Live)

Every cache key **must have a finite TTL**—even if you actively invalidate on writes—as a safety net against missed events or network glitches.
- **Cache Avalanche Prevention (TTL Jitter!):** Never load 50,000 catalog items at boot with the exact same `TTL = 3600s`! At `t = 3600s`, all 50,000 keys expire simultaneously and flatten PostgreSQL. Always add **random jitter**:
  $$\text{TTL} = \text{baseTTL} + \text{random}(0,\ 0.2 \times \text{baseTTL})$$

---

#### 11. Cache Stampede

A **Cache Stampede (Thundering Herd)** occurs when **one ultra-hot key** (e.g., Flash Sale homepage config or trending product) expires, and **2,000 concurrent threads** hit a cache miss simultaneously and launch 2,000 identical heavy DB queries—crashing the database!

##### 3 Production Defenses Against Cache Stampedes
1. **Singleflight / Distributed Mutex Lock:** On miss, acquire a quick Redis lock (`SET lock:product:1 uuid NX PX 3000`). Only **1 winning thread** queries DB and updates Redis; the other 1,999 threads sleep `50ms` and retry reading Redis (or read a stale fallback).
2. **Probabilistic Early Expiration (XFetch) / Background Refresh-Ahead:** When a key's remaining TTL drops below 10%, one background thread asynchronously refreshes it *before* it expires so user requests never see a miss.
3. **Never Evict / Stale-While-Revalidate:** Store `{data, logicalExpireAt}` with a longer physical Redis TTL; if `now > logicalExpireAt`, return stale data immediately to the user while 1 async worker refreshes in the background!

---

#### 12. Cache Penetration

**Cache Penetration** happens when malicious attackers or buggy clients repeatedly request **keys that do NOT exist in either Cache or DB** (`GET /products/-9999` or millions of random fake UUIDs):
- Every request misses Redis AND queries PostgreSQL (`SELECT ... WHERE id = ...` returning 0 rows)—completely bypassing the cache and saturating the DB!

##### 3 Defenses Against Cache Penetration
1. **Cache Null / Missing Markers:** If DB returns empty, store a sentinel `"__NULL__"` in Redis with a **short TTL (`30–60s`)** so repeated scans hit Redis instead of PostgreSQL.
2. **Bloom Filter:** Maintain a compact in-memory or Redis Bloom Filter (`BF.EXISTS`) of all valid existing IDs—reject non-existent IDs in $O(1)$ nanoseconds without touching DB (0% false negatives!).
3. **Strict Input Validation & Rate Limiting:** Reject malformed IDs (non-UUIDs, negative IDs) at the Gateway/Controller before hitting cache/DB.

---

#### 13. Cache Consistency

Accept that **Distributed Cache + DB are eventually consistent** unless guarded carefully:
- **Read-Before-Delete Race:** Thread A reads DB (`$10`), pauses; Thread B writes DB (`$20`) and deletes cache; Thread A wakes up and writes stale `$10` to cache!
  - *Mitigations:* Short safety-net **TTLs**, **Delayed Double-Delete** (delete before/after write + async delete 500ms later), or **CDC-driven cache invalidation (Debezium WAL → Kafka → Cache Updater)** with version timestamps (`if version > cachedVersion`).
- **Golden Rule:** Never rely on a cache as the authoritative source of truth for **money balances or remaining inventory during checkout**—always enforce atomic DB updates (`WHERE qty >= :requested`)!

---

### 9.3 Performance Engineering

---

#### 1. Database Connection Pooling

Opening a fresh PostgreSQL TCP + TLS + backend fork connection per query takes ~20–50ms and destroys database memory. **HikariCP** (Spring Boot default) / **Agroal** (Quarkus default) maintains warm reusable DB connections.

##### HikariCP Sizing Formula (PostgreSQL Rule)
$$\text{Optimal Pool Size} \approx (\text{DB CPU Cores} \times 2) + \text{Effective Spindle Count}$$
- **Counter-Intuitive Production Fact:** On an 8-core DB server, total optimal active connections across the **entire cluster** is roughly **20–40**—NOT 500! Setting `maximum-pool-size: 100` across 20 pods (`2,000` connections!) causes severe DB context switching and lock thrashing.
- **Scale Math:** Keep per-pod pool size small (`10–15`) and place **PgBouncer (Transaction Pooling mode)** or **AWS RDS Proxy** in front of PostgreSQL when scaling to dozens of pods. Always set `connection-timeout: 3000` (fail fast, not 30s!) and `max-lifetime` slightly shorter than DB/firewall idle timeout.

---

#### 2. HTTP Connection Pooling

Reuse persistent HTTP/1.1 Keep-Alive or HTTP/2 connections via **Apache HttpClient 5 (`PoolingHttpClientConnectionManager`)** or **Netty `ConnectionProvider`**:
- Eliminates repeated DNS + TCP 3-way handshake + TLS handshakes and prevents `TIME_WAIT` ephemeral socket exhaustion.
- Always tune **`defaultMaxPerRoute`** (e.g., `50–200` per downstream host—Apache defaults to tiny values!) and set idle connection eviction **shorter** than the downstream LB/server keep-alive timeout.

---

#### 3. Thread Pools

- **Tomcat / Servlet Thread Pool (`server.tomcat.threads.max` default `200`):** Handles inbound HTTP requests.
- **Custom `ExecutorService` Anti-Patterns:** Never use `Executors.newFixedThreadPool()` (unbounded `LinkedBlockingQueue` causes OOM!) or `Executors.newCachedThreadPool()` (unbounded thread creation causes OS `OutOfMemoryError: unable to create new native thread`!).
- **Safe Bounded `ThreadPoolExecutor`:** Specify `corePoolSize`, `maxPoolSize`, bounded `ArrayBlockingQueue`, explicit `RejectedExecutionHandler` (`AbortPolicy` / `CallerRunsPolicy`), and meaningful thread name prefixes for `jstack` debugging—or switch blocking I/O services to **Java 21+ Virtual Threads** (`spring.threads.virtual.enabled=true`, remembering **never to pool Virtual Threads**!).

---

#### 4. Request Concurrency

**Little's Law** mathematically links **Throughput ($\lambda$)**, **Average Latency ($W$)**, and **Required Concurrency ($L$)**:

$$L = \lambda \times W$$

- **Example:** At `1,000 RPS` ($\lambda$) with `50ms` average latency ($W = 0.05\text{s}$), your service needs **$1,000 \times 0.05 = 50$ concurrent active threads/requests**.
- **Latency Spike Amplification:** If a downstream DB slows from `50ms` to `2,000ms` ($2\text{s}$), required concurrency jumps **40x to $2,000$ threads**—instantly overflowing Tomcat's 200-thread pool unless bounded by timeouts and circuit breakers!

---

#### 5. CPU-Bound versus I/O-Bound Workloads

| Workload Type | Examples | Optimal Thread Sizing | Best Java Architecture |
| --- | --- | --- | --- |
| **CPU-Bound** | Encryption, hashing (bcrypt), image/video processing, large JSON serialization, ML inference | **$\approx N_{\text{CPU cores}}$** (more threads just add context-switching overhead!) | Dedicated small bounded ForkJoin/ThreadPool |
| **I/O-Bound** | REST/gRPC calls, PostgreSQL queries, Redis lookups, Kafka I/O | **Much higher concurrency** ($\approx N_{\text{cores}} \times (1 + \frac{W_{\text{wait}}}{C_{\text{compute}}})$) | **Java 21+ Virtual Threads** (`RestClient` + JDBC) OR **Reactive Non-Blocking** (Netty `WebClient` / Quarkus Mutiny) |

---

#### 6. Memory Utilization

Sizing JVM microservices cleanly inside Kubernetes containers:
- Never leave `-Xmx` unset or set to `100%` of container memory! A Java process uses **Heap (`-Xmx`)** **PLUS Off-Heap Memory** (Metaspace, JIT Code Cache, Thread Stacks `~1MB × threads`, Netty `DirectByteBuffers`, native glibc/OS overhead).
- **Best Practice:** Set **`-XX:MaxRAMPercentage=70.0`** (e.g., inside a `1Gi` K8s container, JVM heap gets ~`700MB`, leaving ~`300MB` buffer for off-heap + safety margin against **`OOMKilled` Exit Code `137`**), and always match `requests.memory == limits.memory` (`Guaranteed` QoS class for critical JVM services!).

---

#### 7. Garbage Collection Impact

Stop-the-World (STW) GC pauses freeze all application threads, causing sudden P99 latency spikes and false circuit-breaker/readiness failures:
- **G1GC (Default JDK 17/21):** Great balance of throughput and soft pause target (`-XX:MaxGCPauseMillis=100`).
- **Generational ZGC (`-XX:+UseZGC -XX:+ZGenerational` in JDK 21+) / Shenandoah (Red Hat):** Concurrent, sub-millisecond (`< 1ms`) pause times even on large multi-GB heaps—ideal for strict low-tail-latency microservices!
- **Cut Allocation Rate First:** Even the fastest GC suffers if hot endpoints allocate megabytes of garbage per request (avoid excessive JSON logging, giant intermediate lists, uncompiled `Pattern.compile` in loops, and `SELECT *` loading huge blobs).

---

#### 8. Serialization Overhead

JSON parsing/generation via reflection frequently consumes **20%–40% of total CPU** in microservices:
- Use **Protobuf over gRPC** for internal high-throughput East–West microservice calls (3x–10x smaller wire payload, zero reflection parsing).
- For REST/JSON: enable Jackson **`BlackbirdModule`** (or Quarkus build-time reflection-free Jackson), avoid redundant deserialize→reserialize hops at BFF/Gateway layers, and use streaming (`JsonParser`) or pagination for large payloads.

---

#### 9. Network Overhead

- **Co-Locate Chatty Services:** Keep tightly interacting services in the **same Region and Availability Zone** (same-AZ RTT `~0.2–0.5ms`, cross-AZ RTT `~1–2ms` + cloud cross-AZ data transfer fees!).
- **Batching & Compression:** Replace 50 `GET /items/{id}` calls with 1 `POST /items/batch` or GraphQL/BFF aggregation; enable **HTTP/2 multiplexing** and **gzip/Brotli** compression for large external JSON payloads (>1KB).

---

#### 10. Database Indexing

- **B-Tree Composite Index Leftmost Prefix Rule:** An index on `(tenant_id, status, created_at DESC)` accelerates queries filtering by `(tenant_id)`, `(tenant_id, status)`, or `(tenant_id, status, created_at)`—**but CANNOT be used** for `WHERE status = 'PAID'` alone!
- **Partial Indexes:** Create tiny, super-fast indexes only for hot subsets—essential for **Transactional Outbox**:
  `CREATE INDEX idx_outbox_pending ON outbox_events (created_at) WHERE status = 'PENDING';`
- **Covering Indexes (`INCLUDE`):** `CREATE INDEX idx_orders_cov ON orders (customer_id) INCLUDE (status, total_amount)` allows PostgreSQL **Index-Only Scans** without touching heap pages!
- **Over-Indexing Penalty:** Every extra index slows `INSERT`/`UPDATE`/`DELETE` writes and steals RAM buffer cache.

---

#### 11. Query Optimization

1. **Always Run `EXPLAIN (ANALYZE, BUFFERS)`:** Look for `Seq Scan` on large tables, high buffer reads, expensive sort spills to disk, and bad row-count estimates.
2. **Keep Predicates SARGable:** Never wrap indexed columns in functions (`WHERE LOWER(email) = ?` or `WHERE YEAR(created_at) = 2026` disables B-Tree indexes! Use range queries `>= '2026-01-01' AND < '2027-01-01'` or expression indexes).
3. **Select Only Needed Columns / DTO Projections:** Never fetch 40-column `@Entity` graphs with `CLOB` payloads when the endpoint only renders 4 fields.
4. **Use Keyset/Cursor Pagination** instead of deep `OFFSET` scans.
5. **Enable JDBC Batching:** Set `hibernate.jdbc.batch_size=50`, `order_inserts=true`, `order_updates=true` (and use `SEQUENCE` ID generators instead of `IDENTITY` in PostgreSQL so Hibernate can batch `INSERT`s!).

---

#### 12. N+1 Query Problem

The #1 ORM performance killer in Java microservices:
- Code executes **`1` query** to load 100 `Order` rows, then accesses a `@OneToMany(fetch = FetchType.LAZY)` collection `order.getItems()` inside a loop—triggering **`100` extra `SELECT * FROM order_items WHERE order_id = ?` queries** (`1 + N = 101` round trips)!

##### 4 Clean Fixes
1. **`JOIN FETCH` in JPQL:** `SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.customerId = :cid` *(watch out: never combine `JOIN FETCH` on collections with JPA `Pageable` pagination—Hibernate loads all rows into memory and paginates in RAM! Use two queries or `@BatchSize` instead!)*
2. **`@EntityGraph(attributePaths = "items")`** on Spring Data repository methods.
3. **`@BatchSize(size = 50)` / `hibernate.default_batch_fetch_size: 50`:** Reduces 100 `SELECT`s down to 2 `WHERE order_id IN (...)` queries cleanly—works great with pagination!
4. **Flat DTO Projection / Native Query** (`SELECT new OrderSummaryDto(...)`) + assert max SQL query counts in automated integration tests (**Hypersistence Utils / datasource-proxy**)!

---

#### 13. Backpressure

Flow-control mechanism ensuring fast producers do not overflow slow consumers' memory or database pools:
- **Reactive Streams (`request(n)`):** WebFlux / Reactor / SmallRye Mutiny consumers explicitly pull only $N$ items they have capacity to process.
- **Bounded Blocking Queues:** Size internal `BlockingQueue`s explicitly so memory never explodes under burst.
- **Kafka Consumer Pause/Throttling:** Tuning `max.poll.records` and pausing partitions if downstream storage saturates.

---

#### 14. Load Shedding

When incoming demand exceeds maximum capacity, **reject low-priority / excess requests immediately (`503 Service Unavailable`)** at the Gateway or inbound concurrency filter instead of letting queues balloon and crashing all pods (preserving fast P99 latency for admitted tier-0 requests!).

---

#### 15. Performance Profiling

> *"Premature optimization is the root of all evil—always profile under realistic load before tuning."*

- **`async-profiler` (Gold Standard for Java):** Low-overhead (<1%) sampling profiler avoiding JVM **safepoint bias**; generates interactive **Flame Graphs** for **CPU**, **Heap Allocations**, **Lock Contention**, and **Wall-Clock I/O waits** (`./asprof -d 30 -e cpu -f flame.html <pid>`).
- **JDK Flight Recorder (JFR) + JDK Mission Control (JMC):** Built right into the JVM (`jcmd <pid> JFR.start duration=60s settings=profile filename=/tmp/rec.jfr`)—captures GC pauses, thread stalls, socket I/O, and allocations in production safely.
- **Continuous Profiling:** Grafana Pyroscope / Parca / IBM Instana continuously collecting always-on production flame graphs so you can inspect CPU spikes *even after an incident ended*.

---

### 9.4 Performance Testing

---

#### 1. Load Testing

Tests the system at **expected normal and peak production traffic** (e.g., sustained 1,500 RPS for 30–60 minutes) using **k6**, **Gatling**, **JMeter**, or **Locust**:
- **Goal:** Verify latency SLOs (P95/P99), error rate (`< 0.1%`), DB/CPU headroom, and GC stability under realistic day-to-day peak volume.

---

#### 2. Stress Testing

Gradually ramps traffic **beyond expected peak capacity** (`1x → 2x → 3x → 5x`) until the system degrades or breaks:
- **Goal:** Find the exact **breaking point** (which component fails first: DB pool, Tomcat threads, Gateway, Redis?), verify **Circuit Breakers / Rate Limiters / Load Shedding** kick in gracefully, and confirm the system **recovers automatically** when load returns to normal.

---

#### 3. Soak Testing

Also called **Endurance Testing**: runs steady realistic load (`70–85%` of peak) continuously for **8 to 72 hours**:
- **Goal:** Catch slow-burning defects invisible in 15-minute tests—**memory leaks**, HikariCP/HTTP connection leaks, file descriptor leaks, disk/log growth, PostgreSQL table/autovacuum bloat, and subtle thread leaks.

---

#### 4. Spike Testing

Applies a **near-instantaneous vertical traffic surge** (e.g., `200 RPS → 4,000 RPS` in 5 seconds, then back down—simulating flash sales, ticket drops, or breaking news push notifications):
- **Goal:** Measure **HPA autoscaling lag**, JVM cold-start behavior, Token Bucket rate limiting, and **Cache Stampede** protection.

| Test Type | Traffic Shape | Duration | Primary Question Answered |
| --- | --- | --- | --- |
| **Load Test** | Steady normal + expected peak | 30–60 min | *"Do we meet P95/P99 & error SLOs at expected peak?"* |
| **Stress Test** | Stepwise ramp beyond breaking point | 30–60 min | *"Where is our ceiling, what breaks first, and do we recover?"* |
| **Soak Test** | Steady moderate-high load | 8–72 hours | *"Are there memory, socket, FD, or DB bloat leaks over time?"* |
| **Spike Test** | Sudden extreme burst & drop | 5–15 min | *"Do autoscaling, caches, and rate limiters survive instant shocks?"* |

---

#### 5. Throughput Measurement

Measure **Goodput** (successful business requests/sec meeting latency SLA), not raw RPS (`500` errors return in 1ms and artificially inflate RPS!).
- **Avoid "Coordinated Omission" in Load Tools:** Classic closed-loop generators wait for Request 1 to finish before sending Request 2—hiding massive server stalls! Always use **open-model arrival-rate generators** (e.g., k6 `constant-arrival-rate`, Gatling `constantUsersPerSec`) that keep firing at the true target arrival rate even when the server slows down!

---

#### 6. Latency Measurement

- Always record full **P50, P90, P95, P99, and P99.9** percentiles (never mean alone).
- **Mandatory Staging Realism Rules:**
  1. **Production-Sized Test Data:** Never load-test against a staging DB with 500 rows when production has 50 million rows (full-table scans look fast on 500 rows!).
  2. **Warm Up JVM First:** Discard the first 2–5 minutes of metrics while JIT C2 compilation, HikariCP pools, and caches warm up.
  3. **Stub External Paid APIs Realistically:** Mock Stripe/Twilio with realistic injected P50/P99 latency (`150ms–400ms`), never `0ms` instant mocks!

---

#### 7. Capacity Planning

Size compute, databases, and queues systematically using **Little's Law** + empirical single-pod benchmarks:
1. Measure single-pod max **sustainable RPS** at ~**60% CPU** (keeping latency within SLO).
2. Calculate required pods for peak forecast: $\text{Pods} = \lceil \frac{\text{Forecast Peak RPS}}{\text{Single-Pod Sustainable RPS}} \rceil$.
3. Apply **$N + 1$ / Multi-AZ Failover Headroom:** Ensure that if **1 entire Availability Zone (33% of capacity in a 3-AZ cluster)** fails during peak load, remaining pods handle 100% of traffic without exceeding ~80% utilization!
4. Size DB connections (`pods × poolSize`), Kafka partitions (`>= max consumer pods`), Redis RAM (`working set × 2` for replication/RDB fork overhead!), and network egress accordingly.

---

#### 8. Bottleneck Identification

Follow a disciplined **Top-Down Iterative Bottleneck Loop** during performance tests (never change 5 things at once!):
1. **Verify the Load Generator Isn't the Bottleneck** (check load-runner CPU, ephemeral ports, NIC saturation).
2. **Inspect System & Gateway Metrics:** Check K8s CPU CFS throttling, LB limits, network saturation.
3. **Profile Application (`async-profiler` / JFR):** Spot hot serialization methods, lock contention (`synchronized`), or GC pauses.
4. **Check Pools (`HikariCP`, HTTP Pools, Thread Pools):** Look for `pending` / wait times in USE metrics.
5. **Inspect Database & Downstream (`pg_stat_activity`, `EXPLAIN ANALYZE`, Redis slowlog, OTel traces):** Eliminate the #1 slowest constraint, **re-test**, and verify whether throughput rises until the *next* bottleneck surfaces!

---

#### Quick Interview Revision Checklist (Section 9)
- Can you explain **Horizontal vs Vertical scaling**, **Stateless vs StatefulSets**, and why **K8s HPA + slow JVM startup** lags sudden traffic spikes?
- Can you compare **Cache-Aside, Read-Through, Write-Through, and Write-Behind**—and explain why writes should **`DELETE` rather than `UPDATE`** cache keys?
- Can you explain **Cache Stampede** (Mutex/XFetch), **Cache Penetration** (Null caching + Bloom Filter), and **Cache Avalanche** (TTL Jitter)?
- Can you cite the **HikariCP sizing formula**, apply **Little's Law ($L = \lambda \times W$)**, and explain **JVM container sizing (`-XX:MaxRAMPercentage=70.0`)** + **Generational ZGC/Shenandoah**?
- Can you explain the **B-Tree leftmost prefix rule**, diagnose **`EXPLAIN (ANALYZE, BUFFERS)`**, and list 3+ fixes for the **Hibernate N+1 query problem**?
- Can you contrast **Load, Stress, Soak, and Spike testing**, explain **Coordinated Omission**, and walk through **top-down bottleneck identification** using `async-profiler` / JFR?
