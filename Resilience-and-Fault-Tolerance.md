# Microservices Study Notes — Section 4

## 4. Resilience and Fault Tolerance

> **Critical for IBM and Red Hat**

---

### 4.1 Failure Fundamentals

---

#### 1. Partial Failures

In a monolith, a fatal error (like `OutOfMemoryError`) crashes the whole process—everything fails together or succeeds together. In a distributed microservices architecture, a single user workflow touches multiple independent processes over an unreliable network, creating **partial failures**:
- Example: `OrderService` successfully saves an order, calls `PaymentService` which **times out** (did payment charge the card or crash before charging?), and `NotificationService` is temporarily unreachable.
- **Why it is hard:** The caller cannot distinguish between *"network dropped the request before arrival"*, *"downstream crashed mid-execution"*, and *"downstream succeeded, but the response was lost."*
- **Core rule:** Design every cross-service boundary assuming downstream calls *will* fail partially—combine **timeouts, idempotency, retries, circuit breakers, fallbacks, and Sagas/Outbox**.

---

#### 2. Transient Failures

**Transient failures** are temporary, self-healing faults that resolve on their own within milliseconds to a few seconds:
- Brief network packet loss or DNS blip
- Short database failover or temporary connection pool saturation
- A target Kubernetes pod rolling over or hitting a brief JVM Stop-the-World GC pause
- HTTP `429 Too Many Requests` or `503 Service Unavailable` during traffic spikes before HPA finishes scaling up

**How to handle:** Recover automatically using **bounded Retries with Exponential Backoff and Jitter** (only if the operation is safe/idempotent).

---

#### 3. Permanent Failures

**Permanent failures** are deterministic errors where immediately retrying the exact same request will **always fail** (and only wastes CPU, threads, and quota):
- Client errors: `400 Bad Request` (validation failure), `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `422 Unprocessable Entity` (insufficient funds).
- Application bugs: `NullPointerException`, deserialization failure, schema mismatch, missing database column.
- Dead hardware / unrecoverable external account suspension.

**How to handle:** **Never retry**—**fail fast**, return an explicit RFC 9457 error response for synchronous calls, or route the message to a **Dead-Letter Queue (DLQ)** for asynchronous Kafka/RabbitMQ consumers.

---

#### 4. Cascading Failures

A **cascading failure** happens when a slowdown or failure in one non-critical or downstream component triggers a chain reaction that exhausts shared resources (threads, HTTP connections, memory, DB pools) in upstream callers until the entire platform goes down.

```text
1. InventoryDB slows down (missing index) ──► InventoryService threads block (5s per call)
2. OrderService has NO timeout / bulkhead  ──► All 200 Tomcat threads hang waiting on Inventory
3. OrderService stops responding to Login/Checkout/Health checks!
4. API Gateway connection pool fills up waiting on OrderService ──► ENTIRE PLATFORM DOWN!
```

> **Key Insight:** Slowness is often **more dangerous than an instant crash**! An instant crash (`Connection Refused`) fails in 1ms and frees threads immediately; a hanging 30-second downstream service holds caller threads hostage until the entire fleet collapses.

---

#### 5. Single Points of Failure (SPOF)

A **Single Point of Failure** is any component in the request or data path whose failure brings down the entire system:
- Single-instance API Gateway, un-replicated database primary without automatic failover, single-broker Kafka cluster, or a synchronous hard dependency on a non-redundant third-party service.
- **Hidden SPOFs in Microservices:**
  - Centralized **Redis** rate limiter configured as fail-closed.
  - A shared `AuthService` called synchronously on *every* internal microservice-to-microservice hop (fix: verify stateless **JWT** locally via cached JWKS!).
- **Mitigation:** Multi-AZ active-active redundancy, stateless replicas, fail-open fallbacks for auxiliary systems, and asynchronous decoupling.

---

#### 6. Failure Isolation

**Failure isolation** confines faults inside strict **blast-radius boundaries** so an issue in one feature, tenant, or region cannot spill over into others:
- **Process/Container Isolation:** Separate JVMs/Pods with Kubernetes CPU & Memory limits (`OOMKill` in `ReportingService` doesn't kill `CheckoutService`).
- **Data Isolation:** **Database per Service** prevents a runaway reporting query from locking `orders` tables.
- **Concurrency Isolation:** **Bulkheads** allocate separate thread/connection pools per downstream dependency.
- **Cell / Zone / Shard Isolation:** Isolating tenants or regions so a bad payload from Tenant A doesn't exhaust shared worker pools for Tenant B.

---

#### 7. Fault Tolerance

**Fault tolerance** is the ability of a system to continue delivering acceptable service—even when underlying servers crash, networks partition, or downstream dependencies degrade. Fault tolerance is achieved **not** by trying to prevent hardware/network failures (impossible), but by expecting them and building automatic detection, containment, recovery, and degradation.

---

#### 8. Graceful Degradation

**Graceful degradation** keeps core business capabilities running at reduced richness when non-critical dependencies fail:
- **Example (E-Commerce Product Page & Checkout):**
  - If **`RecommendationsService`** fails → show empty recommendations or generic "Popular Items" cached in Redis (still allow adding to cart!).
  - If **`LoyaltyPointsService`** is down → complete checkout and queue points calculation asynchronously via Kafka.
  - If **`TaxProviderAPI`** fails → use local fallback regional tax table temporarily and flag for reconciliation (or only fail if legally required).

---

#### 9. Fail-Fast Principle

The **fail-fast principle** dictates detecting unrecoverable failures or saturated systems **immediately** and returning an error in milliseconds rather than queueing requests or holding threads open on work destined to fail:
- Validates inputs at the Gateway/Controller before opening DB connections.
- Sets tight Connection & Request timeouts instead of waiting 60 seconds.
- Trips **Circuit Breakers** `OPEN` so requests fail locally in microseconds.
- Applies **Load Shedding** (`503`) when request queues exceed SLA latency.

---

#### 10. Resilient System Design

Production microservice resilience requires **layered Defense-in-Depth**—no single pattern is enough on its own:

| Layer | Pattern | Question Answered |
| --- | --- | --- |
| **Inbound Protection** | Rate Limiting, Load Shedding, Request Validation | *"How do I stop traffic floods from drowning me?"* |
| **Resource Partitioning** | Thread-Pool / Semaphore Bulkheads, Separate DB/HTTP Pools | *"How do I stop one slow dependency from eating all my threads?"* |
| **Outbound Call Guards** | Timeouts, Deadline Propagation | *"How long am I willing to wait?"* |
| **Transient Recovery** | Retries + Exponential Backoff + Jitter + Retry Budgets + Idempotency | *"Should I try again safely?"* |
| **Sustained Outage Guard** | Circuit Breaker (Error-rate & Slow-call-rate) | *"When should I stop calling a broken dependency?"* |
| **User Experience Guard** | Fallbacks & Graceful Degradation | *"What useful response can I still give the user?"* |

---

### 4.2 Timeout Management

---

#### 1. Connection Timeout

The maximum time allowed to establish the underlying TCP (and TLS) connection (`connectTimeout`):
- Catches dead hosts, unreachable IPs, dropped `SYN` packets, or stale service registry entries.
- **Best Practice:** Keep short (**`500ms – 2000ms`** within the same cluster/region).

---

#### 2. Read Timeout

The maximum time the client will wait **between receiving two consecutive data packets** (`socketTimeout` / `readTimeout`) once connected:
- Catches downstream services that accepted the TCP socket, then stalled inside a slow SQL query, deadlock, or Stop-the-World GC pause.
- **Pitfall:** Read timeout only measures *packet-to-packet silence*, **not** total response duration!

---

#### 3. Request Timeout

The **total wall-clock deadline** for one outbound HTTP/gRPC call from start to finish (pool acquisition + DNS + TCP/TLS connect + request write + server think time + full response body read):
- Enforced via Netty `HttpClient.responseTimeout(...)`, Apache HttpClient request config, gRPC `withDeadlineAfter(...)`, or Resilience4j **`TimeLimiter`**.

---

#### 4. End-to-End Timeout

The overall SLA deadline across an entire multi-hop request path starting at the API Gateway or edge client:
```text
Client ──(SLA: 2000ms)──► API Gateway ──► OrderService ──► PaymentService ──► Bank API
```
- If the client gives up at `2000ms`, having `PaymentService` continue waiting 10s on `Bank API` wastes CPU, DB connections, and money (and causes orphaned writes!).

---

#### 5. Timeout Propagation

**Timeout propagation** passes the **remaining time budget** downstream across every service hop:
- **How gRPC does it natively:** The client sets `grpc-timeout: 1500m`. When `OrderService` spends 300ms and calls `PaymentService`, the gRPC stub automatically propagates the *remaining* `1200ms` deadline! If `OrderService` already spent 1600ms, the downstream call doesn't even start (`DEADLINE_EXCEEDED`).
- **How REST/HTTP services do it:** Propagate an `X-Request-Deadline` timestamp (epoch ms) or `X-Timeout-Remaining-Ms` header; middleware computes `remaining = deadline - now()`, caps downstream HTTP client timeout AND JDBC query timeout (`Statement.setQueryTimeout`), and aborts immediately if `remaining <= 0`.

---

#### 6. Timeout Budgeting

When designing multi-hop timeouts, **upstream timeouts must strictly exceed the worst-case downstream execution + retries**:

$$\text{Upstream Timeout} > (\text{Downstream Timeout} \times \text{Max Attempts}) + \sum \text{Backoff Delays} + \text{Local Overhead}$$

- **Wrong Config (Common Bug):**
  - `Gateway timeout = 2000ms`, `OrderService → PaymentService timeout = 2000ms` with `3` attempts (`max 6000ms`).
  - If `PaymentService` stalls, `Gateway` drops the user at `2000ms`, while `OrderService` keeps retrying `PaymentService` for 4 more seconds—wasting work and potentially charging the user after they already saw an error!
- **Right Config:** Either give `OrderService` a shorter per-attempt timeout (`500ms` × 2 attempts + `100ms` backoff = `1100ms` < `2000ms` Gateway timeout), OR propagate the Gateway's `2000ms` remaining deadline so `OrderService` aborts retries once budget expires.

---

#### 7. Downstream Timeout Configuration

Never guess arbitrary timeouts like `30s` or `60s`. Derive them systematically from production **Prometheus latency histograms**:
- Measure **P99 / P99.9** latency of the downstream endpoint under healthy peak load.
- Set timeout slightly above healthy P99.9 (e.g., if normal P99 = `120ms` and P99.9 = `350ms`, set call timeout to **`500ms–800ms`**, not `30,000ms`!).
- Differentiate by operation type: fast cache/auth lookup (`100–300ms`), standard DB read/write (`500–2000ms`), external payment processor (`3000–5000ms`).

---

#### 8. Timeout versus Retry

- **Timeout** detects when a call is taking too long and cuts it off.
- **Retry** decides whether to attempt that failed/timed-out call again.
- **Crucial Warning:** Timing out a `POST /payments` call triggers a `SocketTimeoutException`—**you do NOT know whether the payment server committed the transaction before timing out!** Therefore, **only trigger retries after a Read/Request timeout if the target endpoint is strictly idempotent**.

---

#### 9. Preventing Indefinitely Blocked Requests

**Infinite timeouts (`0` or unset) are a ticking time bomb** in production. Verify **every** blocking boundary in your service has an explicit timeout:

| Boundary | Java / Spring Boot / Quarkus Setting to Check |
| --- | --- |
| **HTTP Pool Lease Wait** | `connectionRequestTimeout` (Apache HttpClient) / `pendingAcquireTimeout` (Netty) |
| **HTTP Connect & Read/Response** | `connectTimeout` + `responseTimeout` / `TimeLimiter` |
| **JDBC Pool Checkout** | `spring.datasource.hikari.connection-timeout` (default 30s—lower to `2000–5000ms`!) |
| **SQL Query / Transaction** | `@Transactional(timeout = 5)` + Postgres `statement_timeout` + `jakarta.persistence.query.timeout` |
| **Redis Client** | `spring.data.redis.timeout` (`500ms–1000ms`!) |
| **Kafka Producer** | `delivery.timeout.ms`, `request.timeout.ms`, `max.block.ms` (avoid blocking HTTP request thread if broker is down!) |

---

### 4.3 Retry Mechanisms

---

#### 1. Retry Policies

A **retry policy** explicitly defines:
1. **Max Attempts:** Total attempts allowed (initial call + retries).
2. **Wait / Backoff Strategy:** How long to pause between attempts (fixed vs exponential + jitter).
3. **Predicate / Exception Classification:** Which exceptions and HTTP status codes trigger a retry vs immediate failure.
4. **Fallback / Budget Cap:** What happens if all attempts fail or the retry budget is exhausted.

---

#### 2. Retryable versus Non-Retryable Exceptions

| Retryable (Transient) | Non-Retryable (Permanent / Client Error) |
| --- | --- |
| `ConnectException` (connection never reached app) | `400 Bad Request` / Validation errors |
| `SocketTimeoutException` / `504` *(ONLY if idempotent!)* | `401 Unauthorized` / `403 Forbidden` |
| `502 Bad Gateway`, `503 Service Unavailable` | `404 Not Found` / `422 Unprocessable Entity` |
| `429 Too Many Requests` *(respect `Retry-After`!)* | `409 Conflict` *(unless optimistic lock `@Version` retry)* |
| JPA `OptimisticLockingFailureException` / DB deadlock | `JsonProcessingException`, `IllegalArgumentException`, `NPE` |
| Resilience4j `CallNotPermittedException` *(Circuit OPEN!)* → **NEVER retry!** | Business domain failures (`InsufficientFundsException`) |

---

#### 3. Fixed Interval Retry

Waits the exact same constant delay between every retry attempt (e.g., `500ms → 500ms → 500ms`).
- **Pros:** Predictable, easy to configure.
- **Cons:** Keeps hammering an overloaded downstream service at a constant cadence instead of letting it recover. Only use for very brief local DB optimistic-lock retries (`50ms`, max 2 retries).

---

#### 4. Exponential Backoff

Increases the wait time exponentially after each consecutive failure:

$$\text{delay}_k = \min(\text{maxCap},\ \text{baseDelay} \times \text{multiplier}^{k})$$

- Example (`base = 200ms`, `multiplier = 2`, `cap = 5s`): `200ms → 400ms → 800ms → 1600ms → 3200ms`.
- **Why:** Quickly recovers from 100ms blips on the first retry while backing off aggressively if a service needs seconds to scale up or fail over.

---

#### 5. Exponential Backoff with Jitter

**The Thundering Herd Problem:** If 2,000 clients fail at `t = 0` due to a brief downstream restart and all use *pure* exponential backoff (`200ms`, `400ms`, `800ms`), all 2,000 clients wake up at the **exact same millisecond** and knock the recovering server right back down!

```text
Pure Exponential Backoff (Synchronized Spikes):
Traffic │   █               █                       █
        │   █               █                       █
        └───┴───────────────┴───────────────────────┴──► Time
           200ms           400ms                   800ms

Exponential Backoff + Full Jitter (Smooth Spread):
Traffic │  ▄ ▆ ▃ ▅ ▂      ▃ ▅ ▄ ▂ ▆ ▃           ▂ ▄ ▃ ▅ ▂ ▃
        └──┴──────────────┴─────────────────────┴──────────► Time
```

- **Full Jitter Formula (AWS Standard):**
  $$\text{sleep} = \text{random}(0,\ \min(\text{cap},\ \text{base} \times 2^{\text{attempt}}))$$
- **Resilience4j Configuration:** `enableExponentialBackoff: true` + `enableRandomizedWait: true` (or `IntervalFunction.ofExponentialRandomBackoff(...)`).

---

#### 6. Retry Limits

Every retry policy **must** enforce a strict upper bound (`maxAttempts`):
- **Synchronous User-Facing HTTP/gRPC Paths:** Keep **`maxAttempts = 2` or `3`** (1 original + 1–2 retries) with short backoff (`100–300ms`), otherwise user P99 latency explodes and caller threads stay occupied too long.
- **Asynchronous Background / Kafka Consumers:** Can tolerate higher retry counts across delayed retry topics (`5s → 30s → 5m → 30m`) before moving to a **DLQ**.

---

#### 7. Retry Budgets

Per-request retry limits (`maxAttempts = 3`) are not enough during a real outage—if 100% of calls fail, traffic **triples (300%)** right when the downstream service is dying!

**Retry Budget Pattern** (used in **Envoy**, **Linkerd**, and Twitter Finagle):
- Instead of allowing every request to retry unconditionally, maintain a sliding-window budget capping **total retries across the client instance at e.g. `10%–20%` of total active/normal requests** (plus a small minimum retry allowance per second for low-QPS services).
- If the service normally sends 1,000 RPS and failures surge, at most 200 retry RPS are allowed—preventing a catastrophic retry storm automatically.

---

#### 8. Retry Amplification

**Retry amplification** occurs when **multiple layers in a call chain independently configure retries**:

```text
Browser/Mobile (retries 3x)
   └──► API Gateway (retries 3x)
           └──► OrderService (retries 3x)
                   └──► PaymentService (retries 3x) ──► Bank DB gets 3 × 3 × 3 × 3 = 81x traffic!
```

- Even 2 retries at 4 hops multiplies load by $3^4 = 81\times$!
- **Architectural Rule:** **Retry at ONE layer only** (typically only at the immediate caller of a leaf service—or only at the client/edge for read-only `GET`s—**never** retry at every hop in a deep synchronous chain).

---

#### 9. Retry Storms

A **retry storm** (metastable failure) is a self-sustaining feedback loop:
1. Downstream `Service B` hits 90% CPU and starts timing out on 20% of requests.
2. Callers retry those 20% failed requests, pushing `Service B` to **150% normal load**.
3. `Service B`'s failure rate jumps to 80%, causing callers to retry **almost every request** (**300% load**!).
4. Even after external user traffic drops back to normal, the internal retry backlog keeps `Service B` pinned at 100% saturation indefinitely until traffic is shed or the circuit trips open.

---

#### 10. Safe Retry of Operations

Before retrying any operation, check **when the failure happened** and **whether the operation is idempotent**:
- **Always safe to retry:**
  - `GET`, `HEAD`, `OPTIONS` (read-only/safe methods)
  - `PUT`, `DELETE` (naturally idempotent when updating to absolute state)
  - `ConnectException` / `UnknownHostException` (TCP handshake never succeeded, so zero request bytes reached the server application—note: ensure your client library distinguishes connect failure from read timeout!)
- **Dangerous to retry blindly:**
  - `ReadTimeoutException` / `504` on **`POST /payments`** or **`POST /orders`**! The server may have already charged the card or inserted the order and crashed while returning the `201 Created` response.

---

#### 11. Retry and Idempotency Relationship

> **Golden Rule:** **Retries and Idempotency are inseparable.** Never enable automatic retries on a state-mutating endpoint or message consumer unless the receiver guarantees **idempotency**.

- **Synchronous APIs:** Caller generates a unique UUID **`Idempotency-Key`** *once before the retry loop* and sends the **same key** on every retry attempt. Receiver stores `(idempotencyKey, requestHash, cachedResponse)` atomically in DB/Redis and returns the cached response on duplicates instead of re-executing.
- **Asynchronous Messaging (Kafka/RabbitMQ):** Consumers use **at-least-once delivery** + retry; every consumer checks `eventId` in an **`inbox_processed_events`** table (or uses natural conditional SQL updates like `UPDATE orders SET status='PAID' WHERE id=? AND status='PENDING'`) inside the same DB transaction.

---

### 4.4 Circuit Breaker

---

#### 1. Why Circuit Breakers Are Needed

Retries handle brief transient glitches, **not** sustained outages. When a downstream service is down or severely overloaded:
- Continuing to send calls and wait 2–5s for timeouts **exhausts the caller's threads and connection pool** while simultaneously hammering the dying downstream service.
- Inspired by electrical circuit breakers, a software **Circuit Breaker** monitors call outcomes, trips `OPEN` when failure/slow rates cross a threshold, and **fails fast in microseconds** (`CallNotPermittedException`)—protecting caller threads and giving the downstream service breathing room to recover.

---

#### 2. Closed State

- **Default healthy state:** All requests flow normally through the Circuit Breaker to the downstream dependency.
- Simultaneously, the Circuit Breaker records each call's outcome (Success, Failure, or Slow Call) inside a **Sliding Window**.
- Once `minimumNumberOfCalls` is reached, if `failureRate >= failureRateThreshold` **OR** `slowCallRate >= slowCallRateThreshold`, the breaker transitions to **`OPEN`**.

---

#### 3. Open State

- **Fail-Fast State:** Every incoming call is **immediately rejected** with `CallNotPermittedException` (or routed straight to a **Fallback**) without touching the network!
- Stays `OPEN` for a configured **`waitDurationInOpenState`** (e.g., `30s`).
- Protects both caller resources (zero blocked threads) and the struggling downstream service (zero incoming load).

---

#### 4. Half-Open State

- **Recovery Probe State:** Once `waitDurationInOpenState` expires, the Circuit Breaker moves from `OPEN` to **`HALF_OPEN`**.
- Permits a small, bounded batch of trial calls (`permittedNumberOfCallsInHalfOpenState`, e.g., `5` calls) through to test whether the downstream dependency has healed (extra concurrent calls beyond those 5 are still rejected!).
- **Evaluation:**
  - If failure rate and slow-call rate of those trial calls are **below** thresholds → transitions back to **`CLOSED`** (normal traffic resumes!).
  - If failure rate or slow-call rate **meets/exceeds** threshold → trips immediately back to **`OPEN`** for another `waitDurationInOpenState`.

---

#### 5. State Transitions

```text
               failureRate >= threshold
               OR slowCallRate >= threshold
        ┌──────────────────────────────────────┐
        │     (after minimumNumberOfCalls)     │
        ▼                                      │
  ┌──────────┐                          ┌──────┴───┐
  │  CLOSED  │                          │   OPEN   │
  └─────▲────┘                          └──────┬───┘
        │                                      │
        │  probe failure/slow rate             │ waitDurationInOpenState
        │  < threshold                       │ expires
        │                                      ▼
        │                             ┌────────────────┐
        └─────────────────────────────┤   HALF_OPEN    │
                                      └────────┬───────┘
          probe failure/slow rate >= threshold │
          (loops back to OPEN) ────────────────┘
```

- **Special Admin States (Resilience4j):**
  - `DISABLED`: Always allows calls (bypasses breaker metrics/blocking—useful in emergencies).
  - `FORCED_OPEN`: Manually locks breaker `OPEN` (e.g., during planned downstream maintenance).

---

#### 6. Failure-Rate Threshold

- **`failureRateThreshold` (default `50%` in Resilience4j):** Percentage of failed calls in the sliding window that triggers `CLOSED → OPEN`.
- **Critical Configuration Rule — `ignoreExceptions` vs `recordExceptions`:**
  - **Do NOT trip a Circuit Breaker on 4xx business/client errors!** If 60 users submit invalid promo codes (`400 Bad Request` / `422` / `EntityNotFoundException`), the downstream service is *healthy*—tripping the circuit breaker would block valid paying users!
  - Record only infrastructure/server failures (`IOException`, `TimeoutException`, HTTP `5xx`, `CallNotPermittedException` disabled).

---

#### 7. Slow-Call-Rate Threshold

- **`slowCallDurationThreshold`** (e.g., `2000ms`) + **`slowCallRateThreshold`** (e.g., `50%`–`80%`).
- **Why it is vital:** If `InventoryService` starts taking `4.5s` to respond with `200 OK` (missing DB index or GC thrashing), **zero exceptions are thrown**—a failure-only circuit breaker never trips! Meanwhile caller Tomcat threads fill up waiting `4.5s` per request.
- Measuring slow calls trips the circuit breaker **early** on latency degradation before full thread exhaustion occurs.

---

#### 8. Sliding Window

Instead of resetting error counts abruptly every minute (which creates edge spikes), Resilience4j continuously tracks outcomes in a **Sliding Window**:
- Two implementations: **Count-Based** and **Time-Based**.

---

#### 9. Count-Based Sliding Window

- Aggregates outcomes of the **last $N$ calls** (e.g., `slidingWindowSize: 100`) using an $O(1)$ circular bit/bucket array.
- Whenever call #101 completes, call #1 is evicted from the window and stats are recalculated in constant time.
- **Best for:** Steady, moderate-to-high traffic services where you want a predictable statistical sample size regardless of time.

---

#### 10. Time-Based Sliding Window

- Aggregates outcomes of all calls made during the **last $N$ seconds** (e.g., `slidingWindowSize: 60` seconds), stored as $N$ 1-second partial aggregation buckets.
- As each second passes, the oldest 1-second bucket expires.

| Aspect | Count-Based Window (`COUNT_BASED`) | Time-Based Window (`TIME_BASED`) |
| --- | --- | --- |
| **Window Size Meaning** | Last $N$ calls (`100` calls) | Last $N$ seconds (`60` seconds) |
| **Low-Traffic Behavior** | Old stale errors from 2 hours ago stay in the window until $N$ new calls arrive! | Old errors naturally age out after $N$ seconds |
| **Burst-Traffic Behavior** | Reacts very fast when 100 failed calls happen within 1 second | Evaluates continuously across all calls inside the time window |

---

#### 11. Minimum Number of Calls

- **`minimumNumberOfCalls` (e.g., `10` or `20`):** Minimum calls required in the sliding window before failure/slow-call rates are calculated.
- Prevents noisy false trips at startup or low traffic: if the very first 1 request fails (`1/1 = 100%`), the breaker waits until at least `minimumNumberOfCalls` calls occur before deciding to trip `OPEN`.
- *Note for `TIME_BASED`:* If only 3 calls occur within 60s and `minimumNumberOfCalls = 10`, the breaker will not trip even if all 3 fail. Tune `minimumNumberOfCalls` appropriately for low-QPS services.

---

#### 12. Wait Duration in Open State

- **`waitDurationInOpenState` (e.g., `15s`–`60s`):** How long the Circuit Breaker remains `OPEN` rejecting calls before entering `HALF_OPEN`.
- Set long enough for a restarting pod or overloaded DB to recover, and short enough to resume traffic promptly once healed.
- Resilience4j also supports **exponential backoff for open wait duration** (`enableExponentialBackoff`) if a dependency keeps failing across repeated `HALF_OPEN` probes.

---

#### 13. Permitted Calls in Half-Open State

- **`permittedNumberOfCallsInHalfOpenState` (e.g., `5` or `10`):** Exact number of test calls allowed in `HALF_OPEN`.
- **Gotcha:** Keep this small (`3–10`). If `permittedNumberOfCallsInHalfOpenState` is too large (e.g., 100), a crashing service gets slammed with 100 simultaneous requests the instant it tries to recover; if it's `1`, one random glitch bounces it back to `OPEN`.
- Also set **`maxWaitDurationInHalfOpenState`** as a safety net so the breaker doesn't get stuck indefinitely in `HALF_OPEN` if probe calls hang or traffic stops.

---

#### 14. Circuit Breaker versus Retry

| Dimension | Retry | Circuit Breaker |
| --- | --- | --- |
| **Problem Addressed** | Brief, isolated **transient failures** | Sustained **outages or severe degradation** |
| **Behavior on Failure** | **Tries again** (increases attempts) | **Stops trying** (fails fast immediately) |
| **Impact on Downstream** | Adds extra load during errors | Relieves load so downstream can recover |
| **Stateful Across Calls?** | Stateless (per single request) | **Stateful** (tracks shared sliding window across all requests) |

---

#### 15. Circuit Breaker Implementation Using Resilience4j (and Quarkus SmallRye)

##### Spring Boot 3 + Resilience4j (`application.yml` & Java)
```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowType: TIME_BASED
        slidingWindowSize: 30
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 60
        waitDurationInOpenState: 20s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignoreExceptions:
          - com.example.order.BusinessValidationException
```

```java
@Service
public class PaymentGatewayClient {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentService")
    public PaymentResult authorize(Order order) {
        return restClient.post().uri("/payments")
            .body(order).retrieve().body(PaymentResult.class);
    }

    // Fallback signature must match return type + args + Throwable
    private PaymentResult paymentFallback(Order order, Throwable ex) {
        return PaymentResult.queuedForAsyncProcessing(order.getId());
    }
}
```

##### Red Hat / Quarkus Equivalent (MicroProfile / SmallRye Fault Tolerance)
In **Quarkus / Red Hat Runtimes**, annotations follow the Eclipse MicroProfile Fault Tolerance spec:
```java
@CircuitBreaker(
    requestVolumeThreshold = 10,
    failureRatio = 0.5,
    delay = 20000, // 20s in open state (ms)
    successThreshold = 5
)
@Timeout(2000)
@Retry(maxRetries = 2, delay = 200, jitter = 100)
@Fallback(fallbackMethod = "paymentFallback")
public PaymentResult authorize(Order order) { ... }
```

---

### 4.5 Bulkhead

---

#### 1. Bulkhead Pattern

Named after watertight compartments in a ship's hull: if one compartment floods, watertight bulkheads isolate the damage so the entire ship does not sink.

In microservices, **Bulkhead** partitions concurrency, threads, or connection pools **per downstream dependency** (or per tenant/endpoint) so a slowdown in one dependency cannot consume all threads and starve unrelated endpoints.

```text
Incoming Traffic (200 Tomcat Threads Total)
   ├──► [Bulkhead: PaymentService]     (Max 30 concurrent) ──► Slow! (Only 30 threads used)
   ├──► [Bulkhead: InventoryService]   (Max 40 concurrent) ──► Still Fast!
   └──► [Bulkhead: CatalogService]     (Max 50 concurrent) ──► Still Fast!
   (Remaining 80 threads reserved for Auth, Health Checks, Orders)
```

---

#### 2. Thread-Pool Bulkhead

- Assigns a **dedicated bounded `ThreadPoolExecutor` + bounded queue** (`maxThreadPoolSize`, `coreThreadPoolSize`, `queueCapacity`) per downstream dependency.
- Caller thread submits the task (`CompletableFuture`) to the isolated pool and waits (with optional timeout).
- Used when you need **asynchronous execution + strict thread pool separation + `TimeLimiter` timeout cancellation**.

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      recommendationSvc:
        maxThreadPoolSize: 16
        coreThreadPoolSize: 8
        queueCapacity: 20
        keepAliveDuration: 20ms
```

---

#### 3. Semaphore Bulkhead

- Uses an in-memory **counting `Semaphore`** (`maxConcurrentCalls`, `maxWaitDuration`) executing directly on the **calling thread** (no extra thread pool or context switch).
- **Lightweight & Zero Thread-Switch Overhead:** Ideal for non-blocking reactive (`WebClient`), **Java 21 Virtual Threads**, or fast local/HTTP calls where you already enforce HTTP client timeouts!

```yaml
resilience4j:
  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 25
        maxWaitDuration: 0ms # Fail fast immediately if 25 calls are already in flight!
```

---

#### 4. Thread Isolation

| Aspect | Thread-Pool Bulkhead | Semaphore Bulkhead |
| --- | --- | --- |
| **Execution Thread** | Separate worker thread in isolated pool | Caller's current thread |
| **Overhead** | Thread context-switching + heap stack allocation + queue overhead | Near zero (atomic counter decrement) |
| **`ThreadLocal` Context** | **Lost** unless propagated explicitly (`MDC`, `SecurityContext`, OpenTelemetry `Context`) | Preserved automatically on same thread |
| **Timeout Interruption** | Can cancel slow hung blocking calls on separate thread | Relies on underlying socket/reactive timeout |
| **With Java 21 Virtual Threads** | Redundant / not recommended (avoid pooling virtual threads!) | **Preferred!** Pair Semaphore Bulkhead + Virtual Threads |

---

#### 5. Resource Isolation

True Bulkheading extends beyond Resilience4j annotations to **every finite runtime resource**:
1. **Separate HTTP Connection Pools per Route:** Configure `maxConnectionsPerRoute` per downstream service so `ReportService` can't steal sockets from `PaymentService`.
2. **Separate DB Connection Pools (HikariCP):** Give high-priority checkout transactions a dedicated HikariCP pool separate from heavy analytics/export queries (or route analytics to a **Read Replica**).
3. **Separate Kafka Consumers / Worker Pools:** Isolate high-priority `payment-events` consumer threads from low-priority `email-digest-events`.
4. **Kubernetes Pod Resource Limits:** Dedicated CPU/memory limits + separate deployments/namespaces for critical vs batch workloads.

---

#### 6. Queue Capacity

- In Thread-Pool Bulkheads (`queueCapacity`), a small bounded queue absorbs short micro-bursts when all worker threads are momentarily busy.
- **Anti-Pattern — Large or Unbounded Queues (`queueCapacity: 10000`):**
  - Masks overload, spikes tail latency (requests sit 15 seconds in queue before even starting!), and balloons JVM heap memory until OOM.
  - **Rule:** Keep `queueCapacity` small (or `0` with `maxWaitDuration: 0ms` for fast fail-fast APIs).

---

#### 7. Pool Exhaustion

When all `maxConcurrentCalls` (or `maxThreadPoolSize` + `queueCapacity`) are occupied:
- New requests immediately fail fast with **`BulkheadFullException`** (without queueing endlessly or touching the downstream network).
- Either trigger a **Fallback** (cached/default response) or return **`429` / `503 Service Unavailable`**.
- Crucially, **other endpoints and bulkheads inside the same service remain 100% healthy**.

---

#### 8. Bulkhead versus Circuit Breaker

| Dimension | Bulkhead | Circuit Breaker |
| --- | --- | --- |
| **Trigger Mechanism** | **Active concurrency count** (`> maxConcurrentCalls`) | **Historical error rate or slow-call rate** (`> threshold%`) |
| **When It Acts** | **Proactively in real time**, even *before* failures occur (e.g., sudden traffic spike to slow service) | **Reactively**, after `minimumNumberOfCalls` show sustained failures/slowness |
| **Behavior When Tripped** | Admits up to $K$ concurrent calls; sheds call $K+1$ onward | Rejects **100% of calls** while `OPEN` |
| **Why Use Both** | Caps blast radius immediately on slow spikes | Cuts off completely dead dependencies so probes don't keep failing |

---

### 4.6 Other Resilience Mechanisms

---

#### 1. Rate Limiting

Controls the frequency of calls over a time window (`limitForPeriod` per `limitRefreshPeriod`):
- **Inbound Rate Limiting (Server / Gateway Protection):** Protects your service from abusive clients, scrapers, or noisy tenants (`429 Too Many Requests`).
- **Outbound Rate Limiting (Resilience4j `@RateLimiter`):** Throttles your own microservice's outbound calls to an external third-party API (e.g., Stripe, Twilio, legacy mainframe) that enforces a strict SLA like `100 req/sec`, preventing quota bans (`429` penalties).

---

#### 2. Token Bucket Algorithm

The most widely used rate-limiting algorithm (used by API Gateways, AWS, Envoy, Stripe, Resilience4j):

```text
          Refill at steady rate R tokens/sec
                     │
                     ▼
              ┌─────────────┐
              │ ● ● ● ● ●   │  Bucket Capacity = B tokens (allows burst up to B!)
              └──────┬──────┘
   Incoming Request ─┴─► Consumes 1 token ──► Allowed (if >= 1 token) / Rejected 429 (if empty)
```

- **How it works:** A bucket holds up to **$B$ tokens** and refills at a fixed rate **$R$ tokens/sec**. Every request removes 1 token. If empty, request is rejected (`429`).
- **Key Property:** Allows **controlled short bursts** up to capacity $B$ while strictly bounding the long-term average rate to $R$.

---

#### 3. Leaky Bucket Algorithm

```text
   Bursty Requests ─► ┌─────────────┐  Overflow ─► Rejected (429)
                      │ ~ ~ ~ ~ ~ ~ │  Queue / Bucket Capacity = B
                      └──────┬──────┘
                             ▼  Fixed drip rate R req/sec (Smooth Output!)
                       Downstream Service
```

- **How it works:** Incoming requests enter a FIFO queue (bucket) of capacity $B$ and drain ("leak") out at a **strictly constant rate $R$**. If the queue is full, new requests overflow and are dropped.
- **Key Property:** Smooths spiky bursts into a **uniform constant output stream**—ideal for protecting fragile legacy downstream databases or mainframes that crash on micro-bursts.

| Feature | Token Bucket | Leaky Bucket |
| --- | --- | --- |
| **Handles Bursts?** | **Yes**, allows bursts up to bucket size $B$ | **No**, smooths all traffic to constant rate $R$ |
| **Typical Behavior on Excess** | Immediate `429` reject when tokens = 0 | Queues up to $B$ (adds queuing latency), then rejects |
| **Best For** | API Gateways, external user APIs | Protecting fixed-capacity backends / traffic shaping |

---

#### 4. Load Shedding

**Load shedding** is a self-preservation mechanism where a server **proactively drops excess or low-priority incoming work (`503 Service Unavailable`)** when its own local resources saturate—*before* queues grow, latency explodes, and GC collapses the JVM.

##### Practical Load Shedding Techniques
- **Concurrency Limits (Little's Law / Adaptive Concurrency, e.g., Netflix concurrency-limits):** Limit max in-flight requests per pod instead of letting Tomcat queue 1,000 requests.
- **Priority-Based Shedding:** Tag requests with priority (`CRITICAL: checkout/payment` vs `SHEDDABLE_PLUS: recommendations/analytics`). Under 85%+ CPU or latency spikes, shed `SHEDDABLE` requests first so `CRITICAL` traffic succeeds cleanly!
- **LIFO Queueing (CoDel):** When overloaded, serve the *newest* requests from the queue and drop stale requests that have already exceeded the client's timeout budget (so you don't spend CPU serving a user who already left!).

---

#### 5. Backpressure

**Backpressure** is a flow-control signal telling an **upstream producer to slow down** because the downstream consumer cannot process data as fast as it arrives:
- **TCP Layer:** Receiver's TCP receive buffer fills up → advertises `window = 0`, pausing sender socket writes.
- **Reactive Streams (Project Reactor / Spring WebFlux / SmallRye Mutiny):** Consumer explicitly signals demand via `Subscription.request(n)`—upstream publisher never pushes more than $n$ items.
- **Bounded Queues & Blocking:** Bounded `BlockingQueue` (`ArrayBlockingQueue`) blocks producer threads rather than causing `OutOfMemoryError`.
- **Kafka Consumers:** Calling `consumer.pause(partitions)` or reducing `max.poll.records` when downstream DB is slow (while remembering `max.poll.interval.ms` so the consumer isn't kicked out of the consumer group!).

---

#### 6. Fallback Mechanisms

When a Timeout, Circuit Breaker (`CallNotPermittedException`), or Bulkhead (`BulkheadFullException`) rejects a call, a **Fallback** returns an alternative safe result:
1. **Stale / Cached Data Fallback:** Serve last-known good data from local Caffeine cache or Redis (e.g., product catalog, currency exchange rates).
2. **Default / Empty Fallback:** Return a safe neutral value (`[]` for product recommendations, default avatar).
3. **Failover to Secondary Provider:** E.g., primary SMS/Payment gateway down → route to secondary provider (with its own circuit breaker!).
4. **Deferred Async Fallback:** Accept checkout/invoice request, store in Outbox/Kafka, return `202 Accepted` ("Your receipt will arrive shortly").

---

#### 7. Fallback Limitations

Fallbacks are **not magic** and frequently fail in production if misused:
- **Never Hide Failures on Critical Writes:** Do *not* return a fake `"200 OK Payment Confirmed"` from a fallback when `PaymentService` failed! Either return a clear error or explicitly transition order state to `PAYMENT_PENDING_RETRY`.
- **Fallback Calling Another Remote Service:** If your fallback makes another synchronous network call to a backup DB/API, that backup often fails under the sudden surge too (cascading failure #2). Keep fallbacks **local in-memory or local cache** whenever possible!
- **Cold / Untested Code Paths:** Fallbacks rarely execute in normal QA—if a fallback has a bug or relies on an unpopulated cache, you only discover it during a P1 outage unless verified via **Chaos Engineering**.

---

#### 8. Graceful Degradation

Architect your platform into **Service Criticality Tiers** upfront so every engineer knows what can degrade safely:
- **Tier 0 (Must Stay Up / Fail Closed):** Authentication, Cart, Checkout, Payment Authorization, Order Placement.
- **Tier 1 (Degrade with Cache/Reduced Scope):** Search, Product Details, Order History.
- **Tier 2 (Fail Silent / Degrade to Empty):** Personalized Recommendations, Customer Reviews, Recently Viewed, Loyalty Points Preview, Analytics.

---

#### 9. Health Checks

Health endpoints tie resilience into orchestration (**Kubernetes**, **OpenShift**, **Load Balancers**):
- **Liveness Probe (`/actuator/health/liveness` / `/q/health/live`):** Is the JVM alive and free of deadlocks? If failed repeatedly → K8s **restarts** the pod. *Rule: Never depend on external DBs/services.*
- **Readiness Probe (`/actuator/health/readiness` / `/q/health/ready`):** Is the pod initialized and ready to serve traffic (not gracefully shutting down, local DB pool healthy)? If failed → K8s **removes pod IP from `EndpointSlices`** without restarting it.
- **Startup Probe (`/actuator/health/startup` / `/q/health/started`):** Protects slow-booting legacy JVMs from being prematurely killed by liveness probes during startup.
- **Circuit Breaker Health Integration:** Expose circuit breaker states (`CLOSED`/`OPEN`/`HALF_OPEN`) in `/actuator/health` metrics for observability—**never fail ReadinessProbe just because an outbound circuit breaker to an optional service is `OPEN`**, or K8s will pull all your pods out of load balancing!

---

#### 10. Retry and Circuit Breaker Interaction

Understanding **Decorator Execution Order** in Resilience4j (and SmallRye Fault Tolerance) is a **classic senior Java interview question**:

##### Default Resilience4j Aspect Order (Outside → Inside)
```text
Caller
  └──► Retry (outermost)
         └──► CircuitBreaker
                └──► RateLimiter
                       └──► TimeLimiter
                              └──► Bulkhead (innermost)
                                     └──► Target Service Call
```

*(Note: Resilience4j default priority order places `Bulkhead` / `TimeLimiter` / `RateLimiter` / `CircuitBreaker` / `Retry`; specifically `Retry` wraps `CircuitBreaker` by default in Resilience4j!)*

##### Why Order Matters Critically
1. **Correct (`Retry` wraps `CircuitBreaker`):**
   - Attempt 1 fails → **CircuitBreaker records failure #1**.
   - Retry sleeps `200ms`, executes Attempt 2 → fails → **CircuitBreaker records failure #2** (say threshold trips `OPEN`!).
   - Retry attempts Attempt 3 → **CircuitBreaker immediately throws `CallNotPermittedException`**.
   - **CRITICAL:** Add `CallNotPermittedException` to **`retry.ignoreExceptions`**! Otherwise Retry wastes cycles retrying against an already-`OPEN` circuit breaker!
2. **Wrong (`CircuitBreaker` wraps `Retry` without realizing it):**
   - If Retry runs 3 attempts *inside* CircuitBreaker, the CircuitBreaker only sees **1 failure** instead of 3 (masking degradation) and multiplies latency inside a single breaker call—**OR** if a `TimeLimiter` wraps `Retry`, the total timeout must cover all retries + backoff!
3. **Fallback Placement:** Place `@Fallback` on the **outermost** layer (on the `@Retry` or method itself) so it catches final exhaustion after retries, `CallNotPermittedException`, `BulkheadFullException`, and `TimeoutException` cleanly.

---

#### 11. Preventing Cascading Failures (Master Playbook)

```text
[Client] ──► [API Gateway: Auth, Rate Limit, Load Shedding, End-to-End Timeout]
                 │
                 ▼
         [OrderService]
           ├─ Inbound: Tomcat/Netty concurrency limit + ReadinessProbe
           ├─ Outbound Call to PaymentService:
           │    1. Bulkhead (maxConcurrentCalls = 25)       ──► Isolates threads/pool
           │    2. Request Timeout (800ms) + Propagated Deadline ──► Never block forever
           │    3. Bounded Retry (max 2, Exp Backoff + Jitter, idempotent only!)
           │    4. Circuit Breaker (Failure + Slow-Call rate) ──► Fail fast if unhealthy
           │    5. Fallback / Async Queue / Graceful Degradation
           └─ Non-Critical Side Effects (Email, Loyalty, Analytics)
                ──► Transactional Outbox ──► Kafka (100% Decoupled!)
```

##### 8-Point Anti-Cascade Checklist
1. **No unbounded timeouts anywhere** (HTTP pool lease, connect, read, request, DB HikariCP, JDBC query, Redis).
2. **Isolate resources with Bulkheads** (per-route HTTP pools, separate HikariCP/worker pools, Resilience4j bulkheads).
3. **Never retry non-idempotent writes without an `Idempotency-Key`**, and **retry at only one layer** with Exponential Backoff + Full Jitter + Retry Budgets.
4. **Configure Circuit Breakers for both Failure Rate AND Slow-Call Rate**, excluding 4xx client errors.
5. **Never block Readiness/Liveness probes on downstream microservices**.
6. **Move non-real-time dependencies to asynchronous Kafka/RabbitMQ messaging** with Transactional Outbox and DLQs.
7. **Shed low-priority load (`503`) before queues/memory saturate**.
8. **Validate resilience regularly using Chaos Engineering** (fault/latency injection in staging/prod via Toxiproxy, Chaos Mesh, or Istio fault injection).

---

#### Quick Interview Revision Checklist (Section 4)
- Can you explain **why a slow downstream service is worse than a dead one** (cascading thread exhaustion vs fast connection error)?
- Can you design **Timeout Budgeting** across `Gateway → Service A → Service B` and explain **Timeout Propagation**?
- Can you explain **Exponential Backoff with Full Jitter**, **Retry Budgets**, **Retry Amplification**, and why **Retry + Idempotency** are inseparable?
- Can you draw the **Circuit Breaker state machine** (`CLOSED ↔ OPEN ↔ HALF_OPEN`), explain **Slow-Call-Rate Threshold**, and compare **Count-Based vs Time-Based Sliding Windows**?
- Can you contrast **Thread-Pool Bulkhead vs Semaphore Bulkhead** (especially with Java 21 Virtual Threads & `ThreadLocal` context propagation)?
- Can you compare **Token Bucket vs Leaky Bucket**, define **Load Shedding** and **Backpressure**, and explain **Fallback anti-patterns**?
- Can you state **Resilience4j's execution order (`Retry` outside `CircuitBreaker`)** and why `CallNotPermittedException` must be ignored by `Retry`?
