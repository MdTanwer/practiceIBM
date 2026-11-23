# Microservices Study Notes — Section 14

## 14. Real-World Microservices Scenarios

> **Must practise for interviews** — *You should be able to **design**, **implement**, and **troubleshoot** each of these 20 production scenarios end-to-end.*

---

#### 1. Payment Succeeds, but Order Creation Fails
- **Concepts to Apply:** `Saga`, `Compensating Transactions`, `Semantic Lock`, `Transactional Outbox`
- **The Problem:** The customer's card has been charged (`PaymentDB` committed), then `OrderService` fails validation, crashes, or times out—leaving the customer billed with no order!
- **Design & Implementation:**
  1. **Preferred Ordering (Create `PENDING` Order First):** Create the Order locally in `status = 'PAYMENT_PENDING'` (Semantic Lock) *before* charging the card, then transition to `CONFIRMED` on `PaymentSucceededEvent`—or `CANCELLED` on failure!
  2. **If Payment Already Succeeded Before Failure (Saga Compensation):**
     - **Saga Orchestrator** catches `OrderCreationFailed` (or **Choreography** `OrderService` emits `OrderRejectedEvent` via Transactional Outbox).
     - `PaymentService` executes an **idempotent `RefundPayment` compensating transaction** (`refundKey = "refund-" + paymentId`), appends a credit/refund ledger entry, and transitions Saga state to `COMPENSATED`.
     - Guard against **late-arriving payment replies** after an Order already timed out and cancelled (must automatically trigger refund!).
- **Troubleshooting & Testing:** Write component tests stubbing `OrderService` / `InventoryService` failure after payment `201` and assert with **Awaitility** that `PaymentService` receives the `RefundPayment` compensation command once.

---

#### 2. Customer Sends Payment Request Twice
- **Concepts to Apply:** `Idempotency`, `Idempotency-Key`, `Payload Fingerprint`, `Concurrent Locking`, `Multi-Hop Key Forwarding`
- **The Problem:** Double-click on mobile, slow 4G response drop, or client HTTP retry submits `POST /api/v1/payments` twice—risking charging the customer **$200 instead of $100**.
- **Design & Implementation:**
  1. Client generates a UUIDv4 **`Idempotency-Key`** *once* per checkout intent and sends it identically on retries.
  2. Server computes `SHA-256(method + path + canonicalBody)` and executes atomically in PostgreSQL:
     ```sql
     INSERT INTO idempotency_keys (user_id, idempotency_key, request_hash, status, expires_at)
     VALUES (:uid, :key, :hash, 'IN_PROGRESS', NOW() + INTERVAL '24 hours')
     ON CONFLICT (user_id, idempotency_key) DO NOTHING;
     ```
  3. **If 0 rows inserted (duplicate):** Check existing row:
     - `request_hash != hash` → reject with **`422 Unprocessable Entity`** (key reused with different payload!).
     - `status == 'IN_PROGRESS'` (and `locked_at` recent) → reject concurrent race with **`409 Conflict` / `425 Too Early` (`Retry-After: 1`)**.
     - `status == 'COMPLETED'` → replay exact cached HTTP `201` status & JSON body!
  4. **3-Step Recovery Point with External Gateway (Stripe/Bank):** Save `payment(status='INITIATED', gateway_key=UUID)` first (Tx 1) → call Stripe outside DB Tx with `Idempotency-Key: gateway_key` → update `CAPTURED` + `idempotency_keys = 'COMPLETED'` (Tx 2).
  5. **On transient `500`/`503`:** Delete/clear `IN_PROGRESS` row so legitimate retries aren't blocked!
- **Troubleshooting & Testing:** Run a JUnit **`CountDownLatch` 10-thread concurrency test** firing the same key simultaneously and assert `paymentRepository.count() == 1` and `WireMock verify(1, postRequestedFor(...))`.

---

#### 3. Database Commits, but Event Publication Fails
- **Concepts to Apply:** `Dual-Write Problem`, `Transactional Outbox`, `CDC / Debezium WAL Tailing`, `Polling Publisher`
- **The Problem:** Calling `orderRepo.save(order)` then `kafkaTemplate.send(event)` performs two non-atomic writes across separate systems. If the pod crashes or Kafka blips right after DB commit, the Order exists in `OrderDB` with **zero `OrderPlacedEvent`**—downstream `Payment` and `Inventory` never run!
- **Design & Implementation:**
  1. Insert/update the business entity **AND** insert an `outbox_events` row (`id`, `aggregate_id` as Kafka key, `event_type`, `payload JSONB`, `headers JSONB` with `traceparent`) inside the **EXACT SAME local `@Transactional` DB commit**.
  2. **Outbox Relay (2 Options):**
     - **Option A — Debezium CDC (Preferred):** Tails PostgreSQL **WAL** via Kafka Connect (`Outbox Event Router` SMT) in `<50ms` with zero DB polling load.
     - **Option B — Polling Publisher:** Scheduled worker queries `WHERE status='PENDING' ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED`, sends to Kafka (`acks=all`), and marks `PUBLISHED`.
  3. Clean up via CDC auto-delete or daily partitioned table drops (`DROP PARTITION`) to prevent PostgreSQL `VACUUM` bloat.
- **Troubleshooting & Testing:** Monitor `outbox_pending_count` and `oldest_pending_age_seconds` gauge in Prometheus; alert immediately if CDC connector pauses or lag exceeds 30s.

---

#### 4. Kafka Consumer Processes the Same Event Twice
- **Concepts to Apply:** `Transactional Inbox`, `Deduplication`, `Natural State-Machine Idempotency`
- **The Problem:** Due to At-Least-Once delivery (producer retries, Outbox relay restarts, or Kafka consumer rebalances), `InventoryService` consumes `OrderPlacedEvent(eventId = E1)` twice—deducting 2 units twice (4 total)!
- **Design & Implementation:**
  - **Pattern A — Transactional Inbox (Universal):**
    ```java
    @Transactional
    public void processOrderPlaced(OrderPlacedEvent event) {
        int inserted = inboxRepo.insertIgnore(event.eventId()); // INSERT ... ON CONFLICT DO NOTHING
        if (inserted == 0) return; // Already processed -> safe idempotent no-op!
        inventoryRepo.deductStock(event.items());
    }
    ```
  - **Pattern B — Natural Idempotency (Zero extra table):**
    - Conditional transition: `UPDATE orders SET status = 'PAID' WHERE id = :id AND status = 'PENDING'`.
    - Or versioned upsert: `INSERT ... ON CONFLICT (id) DO UPDATE ... WHERE EXCLUDED.version > table.version`.
- **Troubleshooting & Testing:** Produce the identical `ConsumerRecord` twice in Testcontainers Kafka and assert via `Awaitility` that DB quantity decreased only once and `.DLT` count stays `0`.

---

#### 5. Consumer Crashes After Processing, but Before Acknowledgement
- **Concepts to Apply:** `At-Least-Once Delivery`, `Manual Offset Commit`, `Effectively-Once Processing`, `Transactional Inbox`
- **The Problem:** Consumer reads offset `42`, commits `UPDATE inventory...` in PostgreSQL, then gets **`OOMKilled` 2ms later before calling `ack.acknowledge()`**. When the replacement pod starts, Kafka re-delivers uncommitted offset `42`.
- **Design & Implementation:**
  1. Never set `enable.auto.commit=true` (can commit *before* DB write completes → silent data loss!) and never call `ack.acknowledge()` inside `@Transactional` *before* DB commit succeeds!
  2. Configure **`AckMode.MANUAL_IMMEDIATE`** + commit offset **strictly AFTER** the local DB transaction commits.
  3. Because the pod committed BOTH business state AND **`inbox_processed_events(event_id = 42)`** atomically in PostgreSQL before crashing, the re-delivered offset `42` hits the Inbox primary-key conflict, skips duplicate mutation, and commits the Kafka offset cleanly (**Effectively-Once Processing**!).
- **Troubleshooting & Testing:** Test crash recovery by throwing a simulated exception after DB commit before `ack.acknowledge()`, then triggering a second poll and confirming zero double mutation.

---

#### 6. Inventory Reaches Zero During Concurrent Purchases
- **Concepts to Apply:** `Optimistic Concurrency Control`, `Atomic Conditional SQL`, `Pessimistic Locking`, `409 Conflict`
- **The Problem:** Only **`1` unit** of `SKU-99` remains. User A and User B click "Buy" at the exact same millisecond. Both read `qty = 1`, check `1 >= 1`, and update `qty = 0` (or `qty = -1`)—**overselling the last item to two users!**
- **Design & Implementation (3 Production Approaches):**
  1. **Atomic Conditional Update (Best & Fastest for Inventory!):**
     ```sql
     UPDATE inventory SET available_qty = available_qty - :qty
     WHERE sku = :sku AND available_qty >= :qty;
     -- Check updated row count! 1 row = reserved! 0 rows = Out of Stock (409 Conflict)!
     ```
     *(No read-before-write race, single DB round trip, short row lock only for that instant!)*
  2. **Optimistic Concurrency Control (`@Version`):** Add `@Version Long version` on JPA entity (`UPDATE ... SET qty=?, version=version+1 WHERE id=? AND version=?`). Losing thread gets `OptimisticLockingFailureException` → return `409 Conflict` or bounded retry (2–3 attempts).
  3. **Pessimistic Row Lock (`SELECT ... FOR UPDATE`):** Best for ultra-hot flash-sale SKUs where 90% of concurrent optimistic retries would fail and waste CPU. *(Keep transaction short and lock SKUs in sorted order to avoid deadlocks!)*
- **Troubleshooting & Testing:** Run a multi-threaded `CountDownLatch` test with `qty = 1` and 20 concurrent buyers; assert strictly `1` buyer gets `200/201`, `19` get `409`, and final DB `available_qty == 0` (never negative!).

---

#### 7. One Downstream Service Becomes Slow
- **Concepts to Apply:** `Request Timeout`, `Bulkhead`, `Circuit Breaker (Slow-Call Threshold)`, `Fallback`
- **The Problem:** `FraudCheckService` normally answers in `30ms`, then slows to `15 seconds` due to a DB lock. Without guards, `OrderService`'s 200 Tomcat threads and HTTP connections block waiting 15s each—starving the entire `OrderService`!
- **Design & Implementation:**
  1. **Strict Timeouts:** Set `connectTimeout = 500ms` and `requestTimeout = 800ms` (calibrated from healthy P99.9) so threads never hang 30s.
  2. **Bulkhead:** Apply Resilience4j Semaphore Bulkhead (`maxConcurrentCalls = 25`, `maxWaitDuration = 0ms`) + separate `maxPerRoute` HTTP pool so slow calls to `FraudCheckService` cannot steal threads from `/health` or `/catalog`.
  3. **Circuit Breaker with `slowCallDurationThreshold`:** Configure `slowCallDurationThreshold: 600ms` and `slowCallRateThreshold: 50%`. Even if `FraudCheckService` returns `200 OK`, the Circuit Breaker trips `OPEN` on slowness and routes to **Fallback** (e.g., async manual review queue for low-value orders or fail-fast `503`)!
- **Troubleshooting & Testing:** Verify in Component tests using WireMock `.withFixedDelay(3000)` that calls abort in ~800ms, Bulkhead rejects call #26 immediately, and Circuit Breaker trips `OPEN` on consecutive slow responses.

---

#### 8. One Service Becomes Unavailable
- **Concepts to Apply:** `Graceful Degradation`, `Circuit Breaker`, `Fallback`, `Service Tiers`
- **The Problem:** `RecommendationService` or `LoyaltyPointsService` goes completely down (`503` / `ConnectException`). Should `Checkout` crash with `500 Internal Server Error`? **No!**
- **Design & Implementation:**
  1. Classify dependencies by **Criticality Tier** (**Tier 0:** Auth, Payment, Checkout vs **Tier 2:** Recommendations, Reviews, Loyalty Preview).
  2. When a Tier 2 Circuit Breaker trips `OPEN`, fail fast in `<1ms` (`CallNotPermittedException`) and execute a **local Fallback**:
     - **Cached / Stale Fallback:** Serve last-known catalog/exchange rates from Redis/Caffeine.
     - **Default Empty Fallback:** Return `recommendations: []` so the Product & Checkout pages render cleanly.
     - **Deferred Async Queue:** Accept checkout and emit `OrderCompletedEvent` via Outbox/Kafka so `LoyaltyPointsService` awards points automatically once it comes back up!
- **Troubleshooting & Testing:** Ensure **Readiness Probes NEVER fail** when a non-critical downstream circuit breaker opens, otherwise K8s removes healthy `OrderService` pods from service!

---

#### 9. Multiple Services Fail Together
- **Concepts to Apply:** `Cascading Failure Prevention`, `Retry Amplification Guard`, `Load Shedding`, `Async Decoupling`
- **The Problem:** A slowdown in `InventoryDB` causes `InventoryService` → `OrderService` → `BFF` → `API Gateway` to fail simultaneously, magnified 27x by multi-layer retries!
- **Design & Implementation:**
  1. **Zero Unbounded Timeouts:** Every HTTP client, HikariCP pool, JDBC query (`statement_timeout`), and Redis call has explicit short timeouts + deadline propagation.
  2. **Stop Retry Amplification:** Retry at **one layer only** with `maxAttempts = 2–3`, **Exponential Backoff + Full Jitter**, **Retry Budgets** (≤10–20% retry traffic), and ignore `CallNotPermittedException` + `4xx`.
  3. **Isolate & Shed:** Bulkheads per dependency, Circuit Breakers (`failure + slow-call`), **Priority Load Shedding (`503`)** dropping non-critical browse/analytics traffic before queues saturate, and decoupling non-sync steps via **Kafka**.
  4. **Never Tie Readiness to Downstream Services:** Prevents cluster-wide pod eviction cascades.
- **Troubleshooting & Testing:** Run Chaos Mesh / Toxiproxy GameDays injecting 100% latency on a leaf service while asserting Tier 0 endpoints keep meeting SLOs.

---

#### 10. API Latency Suddenly Increases
- **Concepts to Apply:** `Distributed Tracing`, `RED & USE Metrics`, `Exemplars`, `Flame Graphs`
- **The Problem:** Users report `/checkout` P99 latency jumped from `150ms` to `4.2s` at 10:15 UTC.
- **Design & Implementation (Live Diagnostic Workflow):**
  1. **Check RED & Recent Changes:** Inspect Grafana RED dashboard (`Rate`, `Errors`, `Duration`) and check ArgoCD/CI deploys or feature flag changes around 10:15 UTC.
  2. **Trace Waterfall (Grafana Tempo / Jaeger):** Click a **P99 Exemplar** dot on Grafana to open the exact slow `traceId` waterfall—immediately identify **which hop slowed down**:
     - *Wide SQL span* → Inspect PostgreSQL `pg_stat_activity` + `EXPLAIN (ANALYZE, BUFFERS)` (missing index, table lock!).
     - *Staircase of 50 tiny spans* → Hibernate **N+1 query** or chatty HTTP loop (fix with `JOIN FETCH` / `@BatchSize` / parallel `CompletableFuture`).
     - *Empty gap before child span* → **HikariCP wait**, HTTP pool exhaustion, **K8s CPU CFS throttling** (`container_cpu_cfs_throttled_seconds_total`), or **JVM GC pause** (`jvm_gc_pause_seconds`).
  3. **Pivot to Logs & Profiler:** Filter Loki/OpenSearch by `traceId`, and run **`async-profiler` / JFR** on a hot pod if CPU-bound.
- **Troubleshooting & Testing:** Enforce `datasource-proxy` max-query-count assertions in CI and tail-based OTel sampling (`duration > 1s`) in production so slow traces are always captured.

---

#### 11. Database Connections Are Exhausted
- **Concepts to Apply:** `HikariCP Connection Pool Management`, `Transaction Boundaries`, `PgBouncer`, `pg_stat_activity`
- **The Problem:** Pods log `HikariPool-1 - Connection is not available, request timed out after 30000ms`; requests stall even though DB CPU might be low (or DB rejects new connections `FATAL: sorry, too many clients already`).
- **Design & Implementation:**
  1. **Never Hold `@Transactional` Open Across HTTP / Kafka Calls!** Move all remote REST/gRPC/Stripe calls completely outside DB transactions.
  2. **Right-Size HikariCP Pools:** Keep per-pod pool compact (`maximum-pool-size: 10–15`) following $(\text{DB CPU cores} \times 2) + \text{spindles}$—ensure `max_HPA_pods × pool_size < DB max_connections` (or place **PgBouncer / RDS Proxy** in Transaction Pooling mode in front of PostgreSQL!).
  3. **Set Strict Timeouts & Guardrails:** `connection-timeout: 3000ms`, `@Transactional(timeout = 5)`, PostgreSQL `statement_timeout = 5000ms` and `idle_in_transaction_session_timeout = 10000ms`.
  4. **Close Streams:** Ensure JPA `Stream<T>` / manual `Connection`s always use `try-with-resources`.
- **Troubleshooting & Testing:** Inspect `hikaricp_connections_active`, `pending`, and query `SELECT state, wait_event, query FROM pg_stat_activity WHERE state != 'idle'`. Enable `hikari.leak-detection-threshold: 5000` temporarily to log exact leak stack traces.

---

#### 12. Service Deployment Breaks Existing Clients
- **Concepts to Apply:** `Backward Compatibility`, `Consumer-Driven Contracts (Pact)`, `Expand-and-Contract`, `OpenAPI Diff`
- **The Problem:** Deploying `OrderService v2` renames `totalPrice` → `totalAmount` (or tightens validation / drops a DB column), breaking `MobileBFF`, `BillingService`, or running `v1` pods during rolling rollout!
- **Design & Implementation:**
  1. **Additive-Only Evolution:** Add new optional fields (`totalAmount`), keep deprecated fields populated (`totalPrice`) until consumers migrate; always enable `@JsonIgnoreProperties(ignoreUnknown = true)`.
  2. **Consumer-Driven Contract Tests (Pact / Spring Cloud Contract) + `openapi-diff`:** Block any breaking PR in CI before merge; run `pact-broker can-i-deploy` before production rollout.
  3. **Major Breaking Changes:** Expose `/api/v2/orders` alongside `/v1`, signal retirement via **`Deprecation`** + **`Sunset`** headers, monitor gateway traffic by client key, and retire only when traffic = 0.
  4. **DB Changes:** Use 3-phase **Expand-and-Contract** migrations (never `RENAME COLUMN` or `DROP COLUMN` in one release).
- **Troubleshooting & Testing:** Roll back immediately (`kubectl rollout undo`), verify consumer/provider contracts in CI, and check Gateway `400`/`422`/`500` breakdown by `User-Agent` / `client_id`.

---

#### 13. Two Events Arrive Out of Order
- **Concepts to Apply:** `Kafka Partition Key Ordering`, `aggregateVersion`, `Upcasting / Event Versioning`
- **The Problem:** `OrderShippedEvent` arrives at `NotificationService` or `OrderQueryView` *before* `OrderPlacedEvent` (e.g., due to missing partition key, non-blocking `@RetryableTopic`, or multi-threaded consumer)—causing a stale event to overwrite newer state (`SHIPPED → CREATED`) or fail on missing entity!
- **Design & Implementation:**
  1. **Producer Partition Key:** Always set Kafka record key = **`aggregateId`** (`orderId`) so all events for one entity hit the **exact same partition** in order; keep `enable.idempotence=true` (preserves order across retries up to `max.in.flight=5`) and never change topic partition count casually.
  2. **Consumer Monotonic Version Guard (`aggregateVersion`):** Include `version: 1, 2, 3...` in every event:
     - **Stale Older Event (`event.version <= current.version`):** Ignore safely (no regression!).
     - **Future Gap Event (`event.version > current.version + 1`):** Do **not** blindly overwrite or drop—throw a retryable `MissingPreviousEventException` to retry briefly (`500ms → 2s`) until missing `version - 1` arrives, or store in a pending buffer!
  3. **Event Versioning:** Evolve event schemas cleanly (Full Compatibility: optional fields with defaults) or use versioned types + **Upcasters**.
- **Troubleshooting & Testing:** Write a Testcontainers Kafka test publishing `OrderUpdated(v=3)` before `OrderUpdated(v=2)` and assert projection remains at `v=3`.

---

#### 14. Compensation Transaction Fails
- **Concepts to Apply:** `Saga Recovery`, `Idempotent Compensation`, `Infinite Retry with Backoff`, `Compensation DLQ & Reconciliation`
- **The Problem:** Order creation failed, so the Saga Orchestrator calls `PaymentService.refund(paymentId)`—but `PaymentService` or Stripe returns `503` (or throws a bug). If the Orchestrator gives up, the user **loses their money with a cancelled order**!
- **Design & Implementation:**
  1. **Golden Rule:** Compensating transactions **MUST be idempotent** and **MUST NEVER be abandoned**!
  2. **Crash-Safe Persistent Saga Log:** Persist `status = 'COMPENSATING'` in the Orchestrator DB so pod restarts resume compensation automatically.
  3. **Tiered Automated Retries:** Retry with exponential backoff (`5s → 30s → 5m → 30m`) across non-blocking retry topics.
  4. **Compensation DLQ & Ops Escalation:** After exhausting automated retries (e.g., expired card or permanent API error), route to **`saga-compensation.DLQ`**, fire a **`SEV1/CRITICAL` PagerDuty alert**, surface in an Ops Dashboard for manual replay/refund, and run nightly **financial reconciliation batch jobs** comparing `PaymentDB` captures vs `OrderDB` confirmed orders.
- **Troubleshooting & Testing:** Stub `RefundPayment` to return `503` 3 times then `200 OK` in Component tests and verify Saga transitions `COMPENSATING → COMPENSATED` automatically.

---

#### 15. One Service Receives Excessive Traffic
- **Concepts to Apply:** `Distributed Rate Limiting`, `Token/Leaky Bucket`, `Priority Load Shedding`, `Autoscaling & Caching`
- **The Problem:** A viral flash sale, scraping botnet, or runaway client retry bug floods `CatalogService` or `CheckoutService` with **20x normal RPS**, threatening to OOM and collapse the cluster.
- **Design & Implementation:**
  1. **Edge Rate Limiting (API Gateway):** Enforce distributed **Redis + Lua Token Bucket** limits (`429 Too Many Requests` + `Retry-After`) keyed by `tenantId`, `userId`, `API-Key`, or IP (`WAF` blocks abusive scrapers).
  2. **Traffic Smoothing:** Use **Leaky Bucket** throttling in front of fragile legacy mainframes/DBs.
  3. **Priority Load Shedding (`503`):** Inside the service/Gateway, tag traffic (`CRITICAL: checkout/auth` vs `SHEDDABLE: browse/analytics`). When concurrency/queue latency spikes, shed `SHEDDABLE` requests immediately so `CRITICAL` payments stay fast!
  4. **Absorb Reads & Scale:** Front hot read endpoints with CDN + Redis (protected by **Singleflight Mutex** against Cache Stampedes) and scale pods via **HPA / KEDA** (fast boot via Quarkus/AppCDS).
- **Troubleshooting & Testing:** Validate with **k6 Spike Testing** (`10x` traffic jump in 5s) asserting `429`/`503` protection triggers cleanly and Tier 0 P99 stays within SLO.

---

#### 16. A New Service Needs to Replace a Monolith
- **Concepts to Apply:** `Strangler Fig`, `Anti-Corruption Layer (ACL)`, `CDC (Debezium)`, `Canary Traffic Shifting`
- **The Problem:** A legacy 500k-line monolith is too slow to deploy and scale, so you must extract `BillingService` without halting feature delivery or attempting a risky "Big Bang" rewrite.
- **Design & Implementation (Incremental Playbook):**
  1. Place an **API Gateway / Reverse Proxy** in front of the monolith (routing 100% to monolith initially).
  2. Build the new `BillingService` independently with its **own clean private database** and an **Anti-Corruption Layer (ACL)** translating any legacy monolith models/codes cleanly.
  3. Synchronize shared transition data asynchronously via **Debezium CDC** tailing legacy DB WAL/Binlog (or dual-write/backfill verification).
  4. **Canary Route Shift:** Route `/api/v1/billing/**` on the Gateway: **shadow mirror first → 5% canary → 50% → 100%** to `BillingService`.
  5. Sever remaining monolith foreign keys/calls via **Branch by Abstraction**, flip `BillingService` to sole writer, and retire legacy billing code + tables.
- **Troubleshooting & Testing:** Run automated shadow-traffic parity comparisons (`Monolith response == BillingService response`) before flipping live traffic to 100%.

---

#### 17. Service Restarts During Active Requests
- **Concepts to Apply:** `Graceful Shutdown`, `K8s preStop Hook`, `terminationGracePeriodSeconds`, `PodDisruptionBudget`
- **The Problem:** During a Kubernetes rolling deploy or autoscaling scale-down, in-flight HTTP requests fail with **`502 Bad Gateway` / `Connection Reset`** because K8s `kubelet` sends `SIGTERM` to Java **simultaneously** while `EndpointSlice` removal is still propagating (1–5s) across `kube-proxy` / Envoy!
- **Design & Implementation (4 Mandatory Pieces):**
  1. **Exec-Form PID 1:** `ENTRYPOINT ["java", "-jar", "app.jar"]` so `SIGTERM` reaches the JVM directly (never shell form!).
  2. **K8s `preStop` Wait (`sleep 10`):** Holds the HTTP socket open for 10s after `Terminating` starts so late traffic arriving from stale `kube-proxy`/Gateway caches still succeeds while routes drain!
  3. **Framework Graceful Shutdown:** Spring Boot `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s` (Quarkus `quarkus.shutdown.timeout=30s`) stops *new* requests after `preStop`, drains active HTTP requests, commits processed Kafka offsets, and closes HikariCP cleanly.
  4. **K8s Safety Bounds:** Set `terminationGracePeriodSeconds: 45` (`> 10s + 30s`!), `maxUnavailable: 0`, and attach a **`PodDisruptionBudget` (`minAvailable: 2`)**.
- **Troubleshooting & Testing:** Run a continuous k6 load test (`200 RPS`) while executing `kubectl rollout restart deployment/order-service` and assert **zero `502`/`503`/dropped requests**!

---

#### 18. One Consumer Cannot Process a Message
- **Concepts to Apply:** `Blocking vs Non-Blocking Retries`, `Dead-Letter Queue (DLQ)`, `ErrorHandlingDeserializer`, `Poison Pills`
- **The Problem:** A single corrupted JSON payload or unexpected null ("poison pill") crashes the Kafka consumer infinitely, blocking an entire partition (`lag` climbs to 100k!) and triggering endless `max.poll.interval.ms` rebalance loops!
- **Design & Implementation:**
  1. **Protect Deserialization First:** Always wrap Kafka deserializers with Spring's **`ErrorHandlingDeserializer`** so invalid JSON/Avro fails cleanly into the error handler instead of hanging `poll()` forever.
  2. **Separate Transient vs Permanent Failures:**
     - **Transient (`SocketTimeoutException`, `SQLTransientConnectionException`):** Route to **Non-Blocking Delayed Retry Topics (`@RetryableTopic`: `10s → 1m → 10m`)** so healthy messages on the main partition keep flowing immediately!
     - **Permanent / Poison Pill (`DeserializationException`, `ValidationException`, `NullPointerException`):** Configure `addNotRetryableExceptions(...)` to skip retries and route straight to **`<topic>.DLT`**.
  3. **DLQ Observability & Replay:** Attach diagnostic headers (`x-exception-message`, `original-offset`, `stacktrace`), fire a Grafana/Alertmanager alert whenever DLQ count > 0, inspect/fix payload or consumer bug, and replay DLQ safely through idempotent consumers.
- **Troubleshooting & Testing:** In Testcontainers Kafka, publish 1 invalid-JSON poison pill followed by 5 valid messages; assert the poison pill lands in `.DLT` with exception headers AND all 5 valid messages process immediately!

---

#### 19. A Service Is Scaled to Multiple Replicas
- **Concepts to Apply:** `Statelessness`, `Shared State (Redis/DB/S3)`, `Leader Election / ShedLock`, `Partitioning & Pool Math`
- **The Problem:** Scaling `OrderService` from `1 → 10` pods breaks hidden single-instance assumptions: HTTP sessions fail randomly across pods, local Caffeine caches drift out of sync, local file uploads vanish, and **all 10 pods fire the same `@Scheduled` midnight billing job 10 times!**
- **Design & Implementation (6-Point Multi-Replica Audit):**
  1. **HTTP Sessions & Auth:** Eliminate in-memory `HttpSession` sticky state → use stateless **JWTs** or shared **Redis** session store.
  2. **File Storage:** Never write uploaded files to container disk → stream directly to **S3 / MinIO / GCS**.
  3. **Caching:** Move shared caches to **Redis** (or L1 Caffeine + Kafka/TTL invalidation).
  4. **Singleton Cron / Background Jobs:** Guard `@Scheduled` jobs with **ShedLock** (DB/Redis lock), **Kubernetes `Lease` Leader Election** (with Fencing Tokens), or extract to a single **Kubernetes `CronJob`**.
  5. **Rate Limiting:** Replace local token buckets with **Redis + Lua** distributed rate limiting.
  6. **Scaling Math:** Verify `max_pods × HikariPoolSize < DB max_connections` (or use PgBouncer) and ensure **Kafka topic `partitionCount >= max_consumer_pods`** so extra replicas don't sit idle!
- **Troubleshooting & Testing:** Always test with `replicas >= 2` in staging/CI (never deploy multi-replica code tested only on 1 local container!).

---

#### 20. A Distributed System Needs Consistent Read Models
- **Concepts to Apply:** `CQRS`, `Materialized Views`, `Transactional Outbox + Kafka`, `Eventual Consistency`, `Read-Your-Own-Writes`
- **The Problem:** Under Database per Service, rendering a unified *Order Management Dashboard* (filtering & paginating Orders by `customerName`, `paymentStatus`, and `shipmentDate`) fails with API Composition at scale (can't efficiently join and paginate 100k rows across 4 services in memory!).
- **Design & Implementation:**
  1. **CQRS Write Side (Command Models):** `OrderService`, `CustomerService`, `PaymentService`, and `ShippingService` commit changes to their private DBs + **Transactional Outbox** → Debezium/Kafka publishes `OrderPlaced`, `CustomerUpdated`, `PaymentCaptured`, `OrderShipped`.
  2. **CQRS Read Side (Materialized View Service):** Consumes all 4 topics and upserts flat, indexed documents into a dedicated **Query DB** (PostgreSQL JSONB/indexed table or **Elasticsearch**):
     - Guard against duplicate/out-of-order updates via **Transactional Inbox** and per-source monotonic version checks (`WHERE incoming.version > view.order_version`).
  3. **Handle Replication Lag (`Eventual Consistency`):** Use **Read-Your-Own-Writes** (return newly created Order DTO directly in `POST` response, or pass version token so UI polls briefly until `view.version >= expected`).
  4. **Rebuild & Reconcile:** Support full **Kafka Event Replay** (with new consumer group) to rebuild the Materialized View anytime schema changes, plus periodic reconciliation checks against source-of-truth DBs.
- **Troubleshooting & Testing:** Monitor Kafka Consumer Lag (`kafka_consumergroup_lag`) on the Query service as a Freshness SLI (`lag < 5s`); test end-to-end projection convergence cleanly with **Awaitility**.

---

### Final Interview Answer Framework (4-Step System Design Formula)

When an interviewer at **IBM**, **Red Hat**, or any senior backend loop throws a live scenario from Section 14 at you, answer systematically in **4 crisp steps**:
1. **Identify the Exact Failure Mode & Root Cause** *(e.g., "This is the classic Dual-Write atomicity gap..." or "This is an async Kubernetes EndpointSlice vs SIGTERM race...")*.
2. **State the Core Architectural Pattern & Trade-Offs** *(e.g., Transactional Outbox + Debezium CDC vs Polling; Saga Orchestration vs TCC; Atomic conditional SQL vs OCC `@Version`)*.
3. **Detail Concrete Java / DB / Kafka / K8s Implementation** *(exact table schema/SQL, Spring/Quarkus config, Resilience4j ordering, header propagation)*.
4. **Explain Observability, Testing & Production Recovery** *(exact Prometheus metrics/alerts, `traceId` debugging, `Awaitility`/`CountDownLatch`/Testcontainers verification, DLQ replay & reconciliation)*.
