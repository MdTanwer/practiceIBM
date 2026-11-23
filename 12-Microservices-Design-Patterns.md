# Microservices Study Notes — Section 12

## 12. Microservices Design Patterns

> *For every pattern in system design interviews (IBM / Red Hat), know **1. Problem It Solves**, **2. Implementation**, **3. Limitations & Pitfalls**, and **4. Alternatives**.*

---

#### 1. API Gateway — *Centralized Routing & Cross-Cutting Concerns*

- **Problem It Solves:** Clients (Web, Mobile, Partners) cannot safely call 50+ internal microservice endpoints directly—doing so exposes internal topology, duplicates JWT auth/rate-limiting/CORS/TLS code in every service, and creates chatty multi-round-trip latency over mobile networks.
- **Implementation:** Deploy a single edge entry point (**Spring Cloud Gateway** on Netty, **Envoy**, **Kong**, **Red Hat 3scale**, **AWS API Gateway**) configured with **Route Predicates** (`Path`, `Method`, `Weight`) and **Pre/Post Filters** (JWT JWKS validation, distributed Redis+Lua Token Bucket rate limiting, `traceparent` injection, URI path rewriting).
- **Limitations & Pitfalls:**
  - Can become a **Single Point of Failure (SPOF)** or scaling bottleneck if run on blocking I/O or backed by a fail-closed Redis outage.
  - Putting business logic or heavy multi-service orchestration inside the shared gateway creates a **"Smart Gateway" monolith**.
- **Alternatives:**
  - **Traditional Reverse Proxy / K8s Ingress** (when you only need basic L7 path routing & TLS without API auth/rate-limiting/quotas).
  - **Backend for Frontend (BFF)** (for client-specific UI aggregation behind the gateway).
  - **Service Mesh Ingress Gateway** (Istio Gateway when unified with mesh rules).

---

#### 2. Service Discovery — *Dynamic Service Location*

- **Problem It Solves:** Container/Pod IPs in Kubernetes and autoscaled clouds are ephemeral—restarts, rolling deploys, and autoscaling continuously reassign `IP:Port` pairs, breaking hardcoded service URLs.
- **Implementation:**
  - **Server-Side Discovery (Modern K8s Standard):** Pods pass `readinessProbe` → K8s adds IPs to `EndpointSlices` → `CoreDNS` resolves `payment-service` to `ClusterIP` → `kube-proxy` / eBPF / Envoy routes to healthy pods automatically.
  - **Client-Side Discovery (VM/Legacy Spring Cloud):** Services self-register and send heartbeats to **Eureka** or **Consul**; callers cache healthy IPs locally and load-balance via Spring Cloud LoadBalancer.
- **Limitations & Pitfalls:**
  - **Stale Registrations:** Heartbeat + cache refresh delays (`30–90s` in Eureka, `1–5s` `EndpointSlice` sync in K8s) can route requests to dying pods unless mitigated with **`preStop` sleep hooks**, fast connect timeouts, and passive outlier detection.
  - **CAP Trade-Off:** AP registries (Eureka) risk brief stale reads; CP registries (Consul/etcd) block registrations if quorum is lost.
- **Alternatives:**
  - **Static DNS / VIPs** (for fixed legacy databases or external SaaS endpoints).
  - **Service Mesh (`xDS` Streaming)** (real-time push discovery directly to sidecar proxies).

---

#### 3. Circuit Breaker — *Failure Isolation*

- **Problem It Solves:** When a downstream service is down or severely degraded, continuing to fire calls and wait 2–5s for timeouts exhausts the caller's threads/connection pool and hammers the dying dependency (**cascading failure**).
- **Implementation:** Wrap outbound calls with **Resilience4j `@CircuitBreaker`** (Spring Boot), **MicroProfile/SmallRye `@CircuitBreaker`** (Quarkus), or **Istio `DestinationRule` Outlier Detection**:
  - Tracks outcomes in a Count- or Time-based sliding window.
  - Trips **`CLOSED → OPEN`** if failure rate (`50%`) or **slow-call rate** exceeds thresholds after `minimumNumberOfCalls`.
  - Fails fast (`CallNotPermittedException` → executes **Fallback**) for `waitDurationInOpenState`, then probes via **`HALF_OPEN`**.
- **Limitations & Pitfalls:**
  - Counting **4xx client business errors** (`400`/`404`/`422`) falsely trips the breaker—must configure `ignoreExceptions`.
  - Local in-memory per pod (10 pods evaluate independently unless using Envoy/mesh-level outlier detection).
- **Alternatives:**
  - **Timeouts + Bulkheads + Load Shedding** (needed alongside circuit breakers).
  - **Asynchronous Queuing (Kafka/RabbitMQ)** (completely eliminates synchronous runtime failure dependency).

---

#### 4. Retry — *Recovery from Transient Failures*

- **Problem It Solves:** Distributed networks and pods suffer brief self-healing glitches (packet drops, rolling pod swaps, short DB failovers, `503`/`429`). Retrying automatically turns transient blips into successful responses.
- **Implementation:** Wrap transient operations with **Resilience4j `@Retry`** (or Spring Kafka non-blocking `@RetryableTopic`) configured with:
  - Bounded **`maxAttempts`** (`2–3` for sync APIs)
  - **Exponential Backoff + Full Jitter** (prevents synchronized thundering herds)
  - **Retryable exception allowlist** (ignore `4xx` and `CallNotPermittedException`!) + **Idempotency Keys** (`Idempotency-Key` header).
- **Limitations & Pitfalls:**
  - **Retry Storms & Amplification:** Multi-layer retries (`Gateway 3x → Order 3x → Payment 3x = 27x` load!) can permanently DDoS an overloaded downstream service.
  - **Double Charging / Duplicate Writes:** Retrying non-idempotent `POST` calls after a `SocketTimeoutException` duplicates mutations!
- **Alternatives:**
  - **Circuit Breaker** (stops retrying during sustained outages).
  - **Dead-Letter Queues + Delayed Retry Topics** (for background async workloads instead of blocking HTTP threads).

---

#### 5. Bulkhead — *Resource Isolation*

- **Problem It Solves:** In a shared server process, a single slow downstream dependency (`RecommendationService` hanging for 5s) can hog all 200 Tomcat worker threads or HTTP sockets, starving completely unrelated endpoints like `/login` and `/checkout`.
- **Implementation:** Partition finite concurrency per downstream dependency (or per tenant):
  - **Semaphore Bulkhead (`Resilience4j`):** Caps concurrent in-flight calls on the caller thread (`maxConcurrentCalls: 25`, `maxWaitDuration: 0ms`)—ideal with **Java 21 Virtual Threads** or WebFlux.
  - **Thread-Pool Bulkhead:** Dedicated bounded `ThreadPoolExecutor` + small `queueCapacity` per dependency.
  - Also isolate **HikariCP DB pools**, **HTTP route pools**, and **Kafka consumer pools**.
- **Limitations & Pitfalls:**
  - Static pool caps can leave spare capacity idle on one bulkhead while rejecting traffic on another (**resource under-utilization**).
  - Thread-Pool bulkheads lose `ThreadLocal` context (`MDC`, `SecurityContext`, OTel trace) unless wrapped with Context Propagation.
- **Alternatives:**
  - **Adaptive Concurrency Limits** (dynamically scales concurrency limits based on observed latency gradients).
  - **Cell-Based / Separate Microservice Deployments** (physical process/pod isolation for noisy tenants or heavy workloads).

---

#### 6. Saga — *Distributed Transaction Coordination*

- **Problem It Solves:** With **Database per Service**, you cannot wrap updates across `OrderDB`, `PaymentDB`, and `InventoryDB` in a single `@Transactional` ACID commit, and 2PC/XA locking does not scale or work with Kafka/REST APIs.
- **Implementation:** Execute a sequence of **local ACID transactions ($T_1 \dots T_n$)**. If step $T_k$ fails before the **Pivot Transaction**, run **Compensating Transactions ($C_{k-1} \dots C_1$)** in reverse order (`RefundPayment`, `ReleaseInventory`):
  - **Saga Choreography:** Decentralized Kafka domain events (good for 2–4 simple steps).
  - **Saga Orchestration:** Centralized persisted state machine (**Temporal**, **Camunda**, custom orchestrator DB) sending commands & handling timeouts/replies (best for 4+ steps).
- **Limitations & Pitfalls:**
  - **No Isolation (`I` in ACID):** Intermediate uncommitted saga states (`PENDING`) are visible to concurrent requests, risking dirty reads/lost updates unless mitigated with **Semantic Locks** (`PENDING`/`FROZEN`).
  - **Compensation Failures & Late-Reply Races:** Compensations must be idempotent and retried until success (with DLQ escalation).
- **Alternatives:**
  - **TCC (Try–Confirm–Cancel)** (when strong application-level resource freezing is required, e.g., wallets/scarce seats).
  - **2PC / XA** (only inside legacy homogeneous relational ecosystems).
  - **Merge Services** (if two entities constantly need tight ACID consistency, they probably belong in one Bounded Context!).

---

#### 7. CQRS — *Separate Read and Write Models*

- **Problem It Solves:** With Database per Service, queries spanning multiple domains (e.g., searching/paginating orders joined with customer names, payment status, and tracking) fail under SQL joins and overwhelm write-optimized transactional schemas.
- **Implementation:** Split **Commands** (write model: PostgreSQL, DDD Aggregates, strong invariants, emits domain events via Outbox) from **Queries** (read model: Kafka consumers updating denormalized flat views in PostgreSQL, Elasticsearch, or MongoDB).
- **Limitations & Pitfalls:**
  - **Eventual Consistency Lag:** A user placing an order may not see it in the Query model for `50ms–2s` unless handled with Read-Your-Own-Writes / UI optimistic states.
  - **Operational Complexity:** Maintaining two schemas, synchronization consumers, and replay pipelines.
- **Alternatives:**
  - **API Composition / Aggregator** (for simple real-time ID lookups without complex cross-service filtering/pagination).
  - **Database Read Replicas** (when all data already resides in one DB and you simply need read scaling).

---

#### 8. Event Sourcing — *Event-Based State Reconstruction*

- **Problem It Solves:** Standard CRUD overwrites rows in place (`UPDATE accounts SET balance = 400`), losing the history of *why* and *how* the state changed, making full auditability, time-travel debugging, and reliable event publishing difficult.
- **Implementation:** Persist **every domain state transition as an immutable append-only event** (`AccountOpened`, `MoneyDeposited`, `MoneyWithdrawn`) in an **Event Store** (`EventStoreDB`, `Axon`, PostgreSQL append log). Reconstruct current entity state by replaying its event stream (optimizing long streams with periodic **Snapshots**). Pair naturally with CQRS Materialized Views.
- **Limitations & Pitfalls:**
  - **Immutable Schema Evolution:** Old events written 3 years ago cannot be altered—requires **Upcasters** to read legacy versions forever.
  - **GDPR "Right to be Forgotten":** Deleting PII from an immutable log requires **Crypto-Shredding** (encrypt PII fields per user key and destroy the user's encryption key).
  - **Steep Learning Curve & Querying:** You cannot query *"WHERE balance > 1000"* on an event log without CQRS projections.
- **Alternatives:**
  - **State-Based Storage + Transactional Outbox + Audit Log Table** (delivers 90% of benefits with far simpler development!).

---

#### 9. Outbox (Transactional Outbox) — *Reliable Event Publication*

- **Problem It Solves:** Eliminates the **Dual-Write Problem** (saving an entity to PostgreSQL and publishing to Kafka are non-atomic across two separate network systems—causing either missing events on crash or phantom events on DB rollback).
- **Implementation:** Inside **one local `@Transactional` DB commit**, insert/update the domain table (`orders`) **AND** insert an event row into `outbox_events` (`id`, `aggregate_id` as Kafka key, `payload`, `headers` with `traceparent`). An async relay—either **Debezium CDC tailing the DB WAL** or a **`FOR UPDATE SKIP LOCKED` Poller**—publishes rows to Kafka, followed by partitioned/batched cleanup.
- **Limitations & Pitfalls:**
  - Guarantees **At-Least-Once** publishing—relay crashes can emit duplicate events (consumers must handle duplicates!).
  - Uncleaned `outbox_events` tables suffer PostgreSQL `VACUUM` dead-tuple bloat (mitigate with CDC auto-delete or daily table partitions).
- **Alternatives:**
  - **Event Sourcing** (the event log itself is the DB, eliminating dual writes).
  - **Listen-to-Yourself** (publish to Kafka first, consume your own event to update DB—high latency & poor UX!).

---

#### 10. Inbox (Transactional Inbox) — *Duplicate Message Prevention*

- **Problem It Solves:** Because Kafka and Outbox relays guarantee **at-least-once delivery** (due to producer retries, pod crashes before offset commit, and consumer group rebalances), consumers *will* receive duplicate events and risk double-charging or double-deducting stock.
- **Implementation:** Store an `inbox_processed_events (event_id UUID PRIMARY KEY, processed_at)` table in the **consumer's own business DB**. Inside one local `@Transactional` commit: execute `INSERT INTO inbox_processed_events (...) ON CONFLICT DO NOTHING`. If `0` rows inserted → **safe duplicate no-op!** Otherwise apply the business mutation atomically, then manually `ack` the Kafka offset **after** DB commit.
- **Limitations & Pitfalls:**
  - Adds an extra table insert + unique index lookup per message and requires periodic TTL pruning (aligned with Kafka retention).
- **Alternatives:**
  - **Natural / State-Machine Idempotency** (`UPDATE orders SET status='PAID' WHERE id=? AND status='PENDING'` — zero extra table!).
  - **Redis `SET NX EX`** (faster for non-financial streams, but *not* atomic with PostgreSQL).

---

#### 11. Strangler Fig — *Gradual Monolith Migration*

- **Problem It Solves:** Rewriting a large legacy monolith from scratch ("Big Bang Rewrite") stalls product delivery for years and almost always fails when requirements shift mid-rewrite.
- **Implementation:** Incrementally replace features one vertical slice at a time behind a routing proxy/API Gateway:

```text
Step 1: Put Gateway in front      Step 2: Route 1 slice to New Svc     Step 3: Monolith Retired
Client ─► [API Gateway]           Client ─► [API Gateway]              Client ─► [API Gateway]
               │                               ├──► [Orders Svc]                     ├──► [Orders Svc]
               ▼                               └──► [Monolith (Rest)]                ├──► [Billing Svc]
          [Monolith]                                    ▲                            └──► [Catalog Svc]
                                              (CDC / Anti-Corruption Layer)
```
  1. Intercept calls at an **API Gateway / Reverse Proxy**.
  2. Build a new microservice (`OrderService`) alongside the monolith, syncing shared data temporarily via **Debezium CDC** and an **Anti-Corruption Layer**.
  3. Flip Gateway route (`/api/orders/**`) from Monolith → New Microservice (optionally via **Canary** traffic split or dark shadow comparison!).
  4. Repeat until the legacy monolith shrinks to zero and is decommissioned.
- **Limitations & Pitfalls:**
  - Temporary dual-system complexity (running both monolith and extracted services, syncing legacy DB tables during transition).
  - Risk of creating a distributed monolith if you fail to sever foreign keys and shared tables before finishing extraction.
- **Alternatives:**
  - **Modular Monolith Refactoring** (if you don't actually need separate deployments yet).
  - **Branch by Abstraction** (for swapping internal modules *inside* code before extracting over the network).

---

#### 12. Sidecar — *Supporting Functionality Alongside Services*

- **Problem It Solves:** Polyglot microservices (Java, Go, Python, Node) all need cross-cutting infrastructure capabilities—mTLS encryption, log shipping, secret fetching, policy checks—without duplicating and maintaining heavy SDKs in 5 different programming languages.
- **Implementation:** Deploy a co-located helper container inside the **exact same Kubernetes Pod** as the application container, sharing the same `localhost` network namespace, lifecycle, and mounted volumes:

```text
┌───────────────────────────── Kubernetes Pod ─────────────────────────────┐
│  ┌─────────────────────┐         localhost / tmpfs        ┌───────────┐  │
│  │  Java Microservice  │◄────────────────────────────────►│  Sidecar  │  │
│  │     (Port 8080)     │   Logs / Secrets / Proxy HTTP    │  (Envoy / │  │
│  └─────────────────────┘                                  │FluentBit /│  │
│                                                           │VaultAgent)│  │
│                                                           └───────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```
  - Examples: **Envoy proxy** (Istio Service Mesh), **HashiCorp Vault Agent** (injects/rotates secrets into `tmpfs`), **Open Policy Agent (OPA)** sidecar, **Fluent Bit** log shipper.
  - *K8s 1.29+ Native Sidecars:* Define sidecars as `initContainers` with `restartPolicy: Always` so Kubernetes guarantees the Sidecar starts **before** the Java app and stays alive **until after** the Java app finishes graceful shutdown (critical for `Job`s and mTLS!).
- **Limitations & Pitfalls:**
  - Per-pod RAM/CPU overhead multiplied by 1,000 pods (`100MB × 1000 = 100GB` RAM!).
  - Adds a few milliseconds of localhost proxy hop latency; startup/shutdown race conditions if not using K8s Native Sidecar syntax.
- **Alternatives:**
  - **In-Process Shared Library / SDK** (Micrometer, Resilience4j—lower memory overhead in homogeneous Java shops).
  - **Node-Level DaemonSet** (1 Fluent Bit per host node instead of 1 per pod) or **Sidecarless Ambient Mesh / Cilium eBPF**.

---

#### 13. Ambassador — *Proxy-Based Outbound Communication*

- **Problem It Solves:** A legacy or polyglot microservice needs resilient outbound networking—TLS/mTLS upgrades, connection pooling, retries, circuit breaking, cloud IAM DB auth, or Redis/DB sharding—without modifying or recompiling the application's client code.
- **Implementation:** A specialized **Outbound Sidecar** (co-located on `localhost`) that acts as the application's **client-side proxy** for all egress traffic:
  - The app makes a simple local call to `localhost:5432` or `http://localhost:9000`.
  - The Ambassador container transparently handles TLS handshakes, IAM token auth, retries, sharding, or circuit breaking toward the remote dependency.
  - Examples: **Google Cloud SQL Auth Proxy**, **PgBouncer sidecar**, **Envoy egress proxy**, **Twemproxy** (Redis sharding).
- **Sidecar vs Ambassador vs Adapter (Classic K8s Multi-Container Distinction!):**
  - **Sidecar:** General helper extending the app (log forwarder, secret fetcher, bidirectional mesh proxy).
  - **Ambassador:** **Outbound** client proxy simplifying communication *from* the app to external services.
  - **Adapter:** **Inbound/Outbound translator** normalizing heterogeneous output formats (e.g., translating legacy app metrics into standard Prometheus format).
- **Limitations & Pitfalls:**
  - Extra local IPC hop, adds failure/debugging layer if Ambassador crashes or runs out of connections.
- **Alternatives:**
  - **In-Process Client Libraries** (`Resilience4j`, `HikariCP`, Spring `RestClient`).
  - **Centralized Egress Gateway** (for managing external outbound allowlists & static egress IPs cluster-wide).

---

#### 14. BFF (Backend for Frontend) — *Client-Specific Backend*

- **Problem It Solves:** Different user experiences (iOS/Android Mobile vs Desktop Web Dashboard vs Public Partner API) have conflicting payload, bandwidth, auth, and round-trip requirements—forcing one shared API into bloated payloads for mobile or too many calls for web.
- **Implementation:** Build a thin, dedicated backend service per UI experience (`Mobile-BFF`, `Web-BFF`, `Admin-BFF`) owned by that client/product team, sitting **behind** the shared API Gateway:
  - Aggregates and shapes domain responses specifically for that client's screens (and can securely hold OAuth2 `HttpOnly` browser cookies away from JavaScript!).
- **Limitations & Pitfalls:**
  - Duplicated composition/auth code across multiple BFFs if there are 5+ client types.
  - **Anti-Pattern:** Allowing core business rules or persistent databases to creep into BFFs!
- **Alternatives:**
  - **Single General-Purpose API Gateway** (when Web & Mobile share identical screens/payloads).
  - **GraphQL Gateway / Federation** (lets Mobile & Web request exact fields from one schema without separate BFF codebases).

---

#### 15. Aggregator — *Combining Multiple Service Responses*

- **Problem It Solves:** Under Database per Service, rendering one view (e.g., *Order Details* or *Travel Booking Summary*) requires data owned by 3–4 separate microservices (`Orders`, `Customers`, `Payments`, `Shipping`).
- **Implementation:** A single service or endpoint receives 1 client request, invokes multiple downstream services (**Scatter–Gather in parallel** via `CompletableFuture.allOf` / Java 21 Virtual Threads / WebFlux `Mono.zip`, **Chained**, or **Branching**), merges results into one unified DTO, and returns it:
  - Always apply **per-call timeouts**, **circuit breakers**, and **graceful degradation fallbacks** (e.g., omit loyalty points if `LoyaltyService` times out instead of returning `500`!).
- **Limitations & Pitfalls:**
  - **Tail Latency Amplification:** Overall latency is capped by the slowest required downstream service ($\max(L_1, L_2, L_3)$).
  - Cannot efficiently filter, sort, or paginate across 100,000 rows from multiple services in memory.
- **Alternatives:**
  - **CQRS Materialized View** (pre-join event streams into a fast read DB—best for search, dashboards, & pagination).
  - **Client-Side Composition** (micro-frontends fetching their own fragments independently).

---

#### 16. Cache-Aside — *Cache Management*

- **Problem It Solves:** Repeated identical reads (`ProductCatalog`, `UserProfile`, `ExchangeRates`) hammer relational databases unnecessarily, wasting DB CPU and adding read latency.
- **Implementation:** Application manages cache lazily alongside the database:
  - **Read:** Check Redis → Hit returns immediately; Miss reads PostgreSQL, writes to Redis with **`TTL + random jitter`** (prevents Cache Avalanche!), returns result.
  - **Write:** Update PostgreSQL first, then **`DELETE`** the Redis key (never blindly `UPDATE` cache under concurrent writes!).
  - Protect hot keys against **Cache Stampede** (singleflight mutex lock / XFetch) and missing IDs against **Cache Penetration** (cache `"__NULL__"` or use Bloom Filters).
- **Limitations & Pitfalls:**
  - Cold-start cache misses after Redis flush/restart; small race window for stale reads between DB commit and cache delete.
- **Alternatives:**
  - **Read-Through / Write-Through** (cache provider owns DB sync transparently—higher write latency).
  - **Write-Behind** (ultra-fast async writes, risks data loss on cache crash).
  - **Debezium CDC Cache Invalidation** (decouples writers from Redis completely).

---

#### 17. TCC (Try–Confirm–Cancel) — *Distributed Transaction Coordination (With Reservation)*

- **Problem It Solves:** Standard Sagas commit immediately and expose intermediate uncommitted state (no Isolation `I`), which fails for **high-stakes financial transfers, wallets, or scarce inventory** where concurrent users must not double-spend funds or overbook the last item before confirmation.
- **Implementation:** Two-phase **application-level resource reservation** orchestrated across participants, each exposing 3 **idempotent** operations:
  1. **`Try`:** Validate constraints & **freeze** resources (`available_balance -= 100; frozen_balance += 100`, `available_qty -= 2; reserved_qty += 2`).
  2. **`Confirm`:** If ALL `Try` steps succeed, commit frozen resources (`frozen_balance -= 100`)—retried idempotently until success.
  3. **`Cancel`:** If ANY `Try` fails or expires, release frozen resources back to available (`frozen_balance -= 100; available_balance += 100`)—retried idempotently until success.
- **Limitations & Pitfalls:**
  - **High engineering overhead:** Requires designing `frozen`/`reserved` columns and 3 idempotent endpoints (`Try`, `Confirm`, `Cancel`) on every service!
  - Must handle **3 subtle TCC edge cases:**
    1. **Empty Cancel:** `Try` timed out before reaching server, then `Cancel` arrives → `Cancel` must succeed cleanly even if no `Try` reservation exists!
    2. **Hanging / Out-of-Order (`Cancel` arrives before `Try`!):** Store transaction log state `CANCELLED` so a late-arriving `Try` sees it and refuses to freeze!
    3. **Orphaned Freezes:** Attach a **reservation TTL** if the orchestrator crashes permanently.
- **Alternatives:**
  - **Standard Saga** (simpler, preferred when resource freezing isn't strictly required).
  - **Local ACID DB** (keep tightly coupled ledger mutations in one service).

---

#### 18. Leader Election — *Coordinating Distributed Workers*

- **Problem It Solves:** When a microservice scales to 10 stateless pods, certain tasks must execute on **strictly ONE pod at a time** (singleton scheduled cron jobs, polling an unpartitioned legacy feed, single-master shard coordination)—otherwise all 10 pods send duplicate invoices or corrupt shared state!
- **Implementation:** Competing pods attempt to acquire a time-bounded lease from an authoritative consensus store:

```text
  Pod 1 ──┐                                  ┌───────────────────────────┐
  Pod 2 ──┼──(Acquire Lease + Fencing Token)►│ Consensus Store (CP):     │
  Pod 3 ──┘   Only Pod 1 wins!               │ K8s Lease API / etcd /    │
              (Renews heartbeat every 5s)    │ Consul / ZooKeeper / DB   │
                                             └───────────────────────────┘
```
  - **Kubernetes Native (`Lease` API):** Spring Cloud Kubernetes / Fabric8 leader election via `coordination.k8s.io/v1` `Lease`.
  - **Consensus Stores:** **etcd**, **Consul**, or **Apache ZooKeeper** (Raft/Zab).
  - **Database Lock (`ShedLock` / PostgreSQL `pg_advisory_lock`):** Simple lock table (`LOCK_UNTIL`) for singleton scheduled tasks when you already have PostgreSQL!
  - **Mandatory Safety Guard — Fencing Tokens:** Pass a monotonically increasing token (`epoch`) on every lock grant so a leader frozen by a 20s JVM GC pause cannot wake up and overwrite storage after its lease expired and a new leader took over!
- **Limitations & Pitfalls:**
  - **Split-Brain Risk** if backed by non-quorum/AP stores or missing Fencing Tokens during GC pauses.
  - **Failover Delay:** If the leader crashes abruptly (`OOMKill`), workers wait until lease TTL expires before electing a new leader.
- **Alternatives:**
  - **Competing Consumers (Kafka Consumer Groups / RabbitMQ / SQS)** (automatically shards partitions across workers without manual leader election!).
  - **DB Work Queue (`SELECT ... FOR UPDATE SKIP LOCKED`)** or **Kubernetes `CronJob`** (lets K8s schedule one pod cleanly without app-level locking!).

---

#### 19. Anti-Corruption Layer (ACL) — *Protecting Domain Boundaries*

- **Problem It Solves:** When a clean, modern DDD microservice must integrate with a messy legacy monolith, third-party vendor API, or upstream service whose data model uses cryptic codes (`CUST_TYP_03`, flat 80-field tables, XML), directly importing upstream models pollutes and corrupts your microservice's clean domain model!
- **Implementation:** Build a dedicated **Adapter / Translator / Facade layer** at the edge of your microservice that converts messy external DTOs/events into clean internal Value Objects and Domain models (and vice versa):

```text
┌─────────────── Clean Order Microservice ───────────────┐       ┌──────────────────────────┐
│  Order Domain Model  ◄──►  [ Anti-Corruption Layer ]   │◄─────►│  Legacy Mainframe / ERP  │
│  (Order, Money, SKU)       (Adapters & Translators)    │ HTTP/ │  (TBL_ORD_HDR, CUST_CD,  │
│                            • Isolates naming & quirks  │ SOAP/ │   string dates, quirks)  │
│                            • Swappable when legacy dies│ Kafka │                          │
└────────────────────────────────────────────────────────┘       └──────────────────────────┘
```
- **Limitations & Pitfalls:**
  - Extra mapping boilerplate, DTO classes, and slight CPU/latency overhead (keep mapping thin and stateless).
- **Alternatives:**
  - **Conformist Pattern** (adopt upstream model directly when upstream is already clean, standard, and stable—e.g., standardized ISO payment specs).
  - **Open Host Service (OHS) + Published Language** (when you own both sides and can ask the upstream team to expose a clean versioned contract).

---

### Master Pattern Decision Matrix

| Category | Pattern | Core Intent | Primary Alternative(s) |
| --- | --- | --- | --- |
| **Routing & Edge** | **API Gateway** | Edge routing, auth, rate limiting, TLS | Reverse Proxy/Ingress, BFF |
| | **Service Discovery** | Dynamic `IP:Port` lookup | K8s ClusterIP/CoreDNS, Service Mesh xDS |
| | **BFF** | Tailored backend per client UI | GraphQL Gateway, Shared API Gateway |
| | **Aggregator** | Combine multiple service responses | CQRS Materialized View, GraphQL |
| **Resilience** | **Circuit Breaker** | Fail fast on sustained outages | Async Queues, Timeout + Load Shedding |
| | **Retry** | Recover from brief transient glitches | DLQ / Non-Blocking Retry Topics |
| | **Bulkhead** | Isolate threads, pools, & concurrency | Adaptive Concurrency, Separate Cells/Pods |
| **Data & Consistency** | **Saga** | Eventual distributed transactions | TCC, Merge Services, 2PC/XA |
| | **TCC** | Resource-freezing 2-phase Saga | Standard Saga, Local ACID DB |
| | **CQRS** | Separate Write and Read models | API Composition, DB Read Replicas |
| | **Event Sourcing** | Append-only event log as source of truth | State CRUD + Outbox + Audit Table |
| | **Outbox** | Atomic DB write + reliable event emit | Event Sourcing |
| | **Inbox** | Atomic consumer event deduplication | Natural State-Machine Idempotency |
| | **Cache-Aside** | Lazy DB caching + write invalidation | Read/Write-Through, CDC Materialized View |
| **Migration & Domain** | **Strangler Fig** | Incremental monolith replacement | Modular Monolith, Branch by Abstraction |
| | **Anti-Corruption Layer** | Isolate domain from messy external models | Conformist, Open Host Service |
| **Infra & Coordination** | **Sidecar** | Co-located helper container in same Pod | Shared SDK, Node DaemonSet, Ambient Mesh |
| | **Ambassador** | Outbound client proxy on `localhost` | In-process SDK (Resilience4j/Hikari), Egress GW |
| | **Leader Election** | Single active worker across replicas | Kafka Partitioning, `SKIP LOCKED`, K8s `CronJob` |

---

#### Quick Interview Revision Checklist (Section 12)
- For **every pattern**, can you state in 30 seconds: **(1) Problem it solves, (2) How to implement it in Java/K8s, (3) Its biggest pitfall, and (4) Its alternative**?
- Can you clearly distinguish **Sidecar vs Ambassador vs Adapter** containers in a Kubernetes Pod?
- Can you contrast **Saga vs TCC vs 2PC** and explain the **TCC "Empty Cancel" and "Hanging Try"** edge cases?
- Can you explain **Outbox + Inbox** together and compare **Event Sourcing** against **State-Based CRUD + Outbox**?
- Can you explain why **Leader Election needs Fencing Tokens**, and when **Kafka partitioning / `SKIP LOCKED` / K8s `CronJob`** beats Leader Election?
- Can you combine **Strangler Fig + Anti-Corruption Layer + CDC** to answer *"How would you migrate a legacy monolith safely to microservices?"*
