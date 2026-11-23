# Microservices Study Notes — Section 10

## 10. Testing Microservices

### Microservices Testing Strategy (Testing Honeycomb vs Pyramid)

In a monolith, a traditional **Testing Pyramid** works well (mostly Unit tests + a few Integration/UI tests). In microservices, most bugs live **at the integration seams** (JSON serialization, HTTP timeouts, SQL constraints, Kafka headers, Circuit Breakers)—shifting heavy investment toward the **Testing Honeycomb** (Component, Integration, & Contract tests) while keeping slow, brittle **End-to-End (E2E) tests** minimal:

```text
          ▲
         / \         E2E Tests (Few — 3 to 5 Critical Smoke Journeys)
        /───\
       /     \       Contract Tests (Pact / Spring Cloud Contract / OpenAPI)
      /───────\
     /         \     Component & Integration Tests (Testcontainers + WireMock) ◄── Sweet Spot!
    /───────────\
   /             \   Unit Tests (Pure Domain Logic, Value Objects, Aggregates)
  ─────────────────
```

---

#### 1. Unit Testing

Tests **isolated business logic** (DDD Entities, Value Objects, Aggregates, Domain Services, pricing/tax rules) **in-memory** with **JUnit 5 + AssertJ + Mockito**:
- **Speed:** Executes in milliseconds with **zero Spring/Quarkus context boot**, zero Docker containers, and zero network calls.
- **Best Practices:**
  - Test **domain behavior and invariants** (`order.addItem(...)` throws if order is already `PAID`), not private methods or trivial getters/setters.
  - Do not mock Value Objects (`Money`, `Address`) or simple POJOs—construct real instances.

---

#### 2. Integration Testing

Tests **narrow infrastructure boundaries** where your application connects to external drivers, serializers, or frameworks:
- Repository SQL/JPQL queries ↔ Real PostgreSQL (Testcontainers)
- Kafka Producer/Consumer serialization & headers ↔ Real Kafka
- HTTP Client (`RestClient`/`WebClient`) + Resilience4j timeouts ↔ **WireMock**
- **Spring Boot Test Slices (Fast!):** `@WebMvcTest` (controllers + `@Valid` + `ProblemDetail`), `@DataJpaTest` (JPA repositories), `@JsonTest` (Jackson date/money serialization).

---

#### 3. Component Testing

A **Component Test** (Service-Level Black-Box Test) boots **one entire microservice** (`@SpringBootTest(webEnvironment = RANDOM_PORT)` or `@QuarkusTest`) connected to **its own real PostgreSQL & Kafka containers (Testcontainers)** while **stubbing external downstream microservices (WireMock)**:

```text
Test Runner (REST Assured / WebTestClient)
   │
   ▼  Real HTTP + JSON + Auth
┌──────────────────────────────────────────────────────────┐
│            OrderService (Full Spring Boot App)           │
└──────────────┬─────────────────────────────┬─────────────┘
               │ Real JDBC / Kafka           │ Real HTTP + Resilience4j
               ▼                             ▼
  [Testcontainers: Postgres + Kafka]    [WireMock: PaymentSvc]
```

- **Why Component Tests Are the Microservices Sweet Spot:** They test controller validation, Spring Security, `@Transactional` + Outbox, Flyway migrations, Jackson serialization, and Resilience4j timeouts/circuit breakers **all together** in seconds—with **zero dependency on any other team's deployed environment**!

---

#### 4. Contract Testing

Verifies that the **HTTP request/response format** or **async event schema** between a **Provider** and **Consumer** adheres strictly to a shared contract—catching breaking API changes in CI without spinning up a live multi-service cluster.
- **Provider-Driven Contract Testing:** Publish OpenAPI 3.1 / Protobuf / Avro schema, then run **`openapi-diff`** or **Confluent Schema Registry compatibility checks** in CI on every PR.
- **Consumer-Driven Contract Testing (CDC):** Consumers explicitly specify which endpoints, fields, and statuses they rely on.

---

#### 5. Consumer-Driven Contract Testing (CDC)

Tools: **Pact** (polyglot HTTP + Kafka messages) and **Spring Cloud Contract**:

```text
1. Consumer CI (OrderService):
   Runs mock test against Pact MockServer ─► Generates contract JSON ─► Uploads to Pact Broker

2. Provider CI (PaymentService):
   Downloads all Consumer Contracts from Pact Broker ─► Replays against real PaymentController
   ─► Fails PR immediately if PaymentService broke any Consumer expectation!

3. Deployment Gate:
   `pact-broker can-i-deploy` verifies matrix compatibility before Kubernetes rollout!
```

- **Covers Both Sync & Async:** Supports HTTP REST contracts AND **Message Pact** contracts (verifying Kafka event JSON schemas between producers and consumers without even needing a live Kafka broker during contract check!).

---

#### 6. End-to-End (E2E) Testing

Tests a complete user journey across **multiple real, deployed microservices** (API Gateway → Order → Payment → Inventory → Notification → real DBs) in a staging or ephemeral Kubernetes namespace.
- **Why You Must Minimize E2E Tests:**
  - Extremely **slow** (minutes per run), **expensive** to maintain, and **flaky** (one unrelated service deploying breaks everyone's suite; hard to isolate test data).
- **Best Practice:** Keep E2E tests to **3–5 critical smoke journeys** (e.g., *User Login → Add to Cart → Checkout → Payment Confirmation*). Push all edge-case, validation, error-handling, and compensation permutations down to **Component** and **Contract** tests!

---

#### 7. API Testing

Black-box verification of HTTP endpoints: status codes, JSON structure, pagination, filtering, authentication/RBAC (`401`/`403`), Bean Validation (`400`), and RFC 9457 `ProblemDetail` errors.
- **Tools:** **REST Assured** (Java), **Spring `WebTestClient` / `MockMvc`**, **Karate DSL**, **Postman/Newman**.

```java
given()
    .header("Authorization", "Bearer " + validJwt)
    .header("Idempotency-Key", UUID.randomUUID().toString())
    .contentType(ContentType.JSON)
    .body("""
        {"customerId": "c-101", "amount": 49.99, "currency": "USD"}
        """)
.when()
    .post("/api/v1/orders")
.then()
    .statusCode(201)
    .header("Location", matchesPattern("/api/v1/orders/.*"))
    .body("status", equalTo("PENDING"));
```

---

#### 8. Service Virtualization

Simulates complex, rate-limited, paid, or unavailable **external/legacy dependencies** (Stripe/Adyen, Core Banking mainframes, SAP, SOAP endpoints, third-party KYC APIs) with realistic stateful flows, edge-case responses, and latency distributions:
- **Tools:** **WireMock**, **Microcks** (Kubernetes-native mocking directly from OpenAPI, AsyncAPI, gRPC `.proto`, and GraphQL specs—very popular in Red Hat / OpenShift ecosystems!), **Hoverfly**, **Mountebank**.

---

#### 9. Mocking Downstream Services

| Approach | How It Works | What It Misses | When to Use |
| --- | --- | --- | --- |
| **In-Process Mock (`@MockitoBean`)** | Replaces Java client bean (`PaymentClient`) in memory | Skips HTTP serialization, headers, URL paths, connection pool, **timeouts, retries, & circuit breakers!** | Unit tests or narrow controller slice tests (`@WebMvcTest`) |
| **HTTP Wire-Level Stub (`WireMock` / `MockWebServer`)** | Spins up real local HTTP server returning stubbed JSON/faults | Only network hops across physical hosts | **Component & Resilience tests** (proves `RestClient`, Jackson, `TimeLimiter`, `Retry`, & `CircuitBreaker` actually work!) |

```java
wireMockServer.stubFor(post(urlEqualTo("/api/v1/payments"))
    .willReturn(aResponse()
        .withStatus(503)
        .withFixedDelay(2500))); // Triggers real HTTP ReadTimeout / Resilience4j Retry!
```

---

#### 10. Testcontainers

**Testcontainers** spins up **real, ephemeral Docker containers** (**PostgreSQL**, **Kafka / Redpanda**, **Redis**, **Keycloak**, **MongoDB**, **LocalStack**) programmatically from JUnit 5!

- **Why H2 In-Memory DB Is Dangerous:** Tests pass on H2, then **crash in production on PostgreSQL** because H2 lacks `JSONB`, `FOR UPDATE SKIP LOCKED`, `INSERT ... ON CONFLICT`, PostgreSQL dialect casting, and real Flyway migration syntax!
- **Spring Boot 3.1+ `@ServiceConnection` (Zero Boilerplate!):**

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
class OrderComponentTest {

    @Container
    @ServiceConnection // Automatically configures spring.datasource.* URL/user/pass!
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));
}
```
- **Quarkus Dev Services:** Automatically starts Testcontainers databases, Kafka, Redis, and Keycloak during `./mvnw test` and dev mode **with zero test code configuration**!
- **CI Speed Tip:** Declare containers in a shared `static` base class (or Singleton Container pattern) so **all test classes reuse 1 Postgres + 1 Kafka container + 1 Spring `ApplicationContext`** instead of booting Docker per test class!

---

#### 11. Database Integration Testing

Testing persistence directly against the **exact production database engine (PostgreSQL/MySQL) + real Flyway/Liquibase migrations**:
- **What to Verify:**
  1. Flyway/Liquibase DDL scripts apply cleanly from scratch.
  2. Custom native SQL / JPQL queries, `JSONB` filters, `FOR UPDATE SKIP LOCKED`, `ON CONFLICT` upserts, and unique constraints work properly.
  3. **Optimistic Locking (`@Version`):** Simulate two concurrent updates and verify the second throws `OptimisticLockingFailureException` / `409 Conflict`.
  4. **N+1 Query Assertions:** Use `datasource-proxy` / Hypersistence Utils to assert `assertSelectCount(1)` on endpoints so accidental lazy-loading loops fail CI!
- **Critical `@Transactional` Test Pitfall:** Putting `@Transactional` on a `@SpringBootTest(webEnvironment = RANDOM_PORT)` test rolls back the test thread's setup data—meaning the real Tomcat HTTP handler thread **cannot see uncommitted setup rows**, and `@Async`/Kafka consumers can't see them either! For component tests, commit normally and clean tables explicitly (`TRUNCATE ...` or `@Sql` cleanup) before each test.

---

#### 12. Message Broker Integration Testing

Testing Kafka/RabbitMQ interactions against a **real containerized broker (`KafkaContainer` / `RedpandaContainer` / `RabbitMQContainer`)**:
- **What to Verify:**
  1. **Transactional Outbox → Kafka Relay:** Verify `Order` save + `outbox_events` insert commit atomically and publish to the expected topic with `aggregateId` as the partition key.
  2. **Header Propagation:** Assert `traceparent`, `eventType`, and `eventVersion` exist on the Kafka `ConsumerRecord`.
  3. **Poison Pill & DLQ Routing:** Send malformed invalid JSON bytes into Kafka and assert `ErrorHandlingDeserializer` + `DefaultErrorHandler` route the message to `<topic>.DLT` **without crashing the consumer loop**!
  4. **Manual Offset Ack:** Assert failed processing does *not* commit the offset prematurely.

---

#### 13. Failure Injection

Deliberately introducing faults into network sockets, HTTP responses, and databases during automated tests to validate failure paths:
- **WireMock Faults:** `Fault.CONNECTION_RESET_BY_PEER`, `Fault.MALFORMED_RESPONSE_CHUNK`, `Fault.RANDOM_DATA_THEN_CLOSE`, HTTP `503`/`504`, `withFixedDelay(3000)`.
- **Toxiproxy (Testcontainers `ToxiproxyContainer`):** Programmable TCP proxy placed between your microservice and PostgreSQL/Redis/Kafka/HTTP dependencies to inject:
  - Latency + jitter (`2000ms ± 500ms`)
  - Bandwidth throttling (`1 KB/s`)
  - Mid-connection TCP cuts (`cutConnectionUpstream()`) to test HikariCP failover, Kafka consumer rebalance recovery, and Redis fallback behavior!

---

#### 14. Chaos Engineering

**Chaos Engineering** is the discipline of experimenting on distributed systems in staging or controlled production environments to verify that resilience mechanisms work under real-world turbulence (**Principles of Chaos**):

1. **Define Steady State:** Measurable SLIs (e.g., `checkout success rate >= 99.9%`, `P99 <= 500ms`).
2. **Form a Hypothesis:** *"If 1 of 3 `PaymentService` pods is `OOMKilled`—or `RecommendationService` injects 5s latency—`Checkout` steady state remains completely intact."*
3. **Run Controlled Experiments (Small Blast Radius + Automated Kill Switch):**
   - Tools: **Chaos Mesh**, **LitmusChaos**, **Gremlin**, **AWS Fault Injection Service (FIS)**, or **Istio VirtualService fault injection**.
   - Fault types: Pod kill, node drain, network latency/partition, packet loss, CPU/memory stress, DNS failure, clock skew.
4. **Verify & Fix Weaknesses:** If steady state drops (e.g., an overlooked missing timeout caused cascading thread starvation!), abort immediately, fix the architecture, and re-verify during GameDays.

---

#### 15. Resilience Testing

Automated tests inside CI proving that **Resilience4j / SmallRye Fault Tolerance** configs are wired properly and actually trigger:
- **Timeout Test:** Stub WireMock with `3000ms` delay → assert client aborts at configured `500ms` timeout (never hangs 30s!).
- **Retry Test:** Stub WireMock `503` twice then `200 OK` → assert final result succeeds AND `wireMock.verify(3, postRequestedFor(...))`; stub `400 Bad Request` → assert **`verify(1, ...)`** (zero retries on non-retryable 4xx!).
- **Circuit Breaker State Transition Test:** Send `minimumNumberOfCalls` failing `503` responses → assert `circuitBreaker.getState() == State.OPEN`, then send 5 more requests → assert all 5 execute Fallback **AND WireMock receives `0` additional HTTP calls**!
- **Bulkhead Concurrency Test:** Fire 20 parallel threads against a slow WireMock endpoint with `maxConcurrentCalls = 5` → assert 5 proceed and 15 fail fast with `BulkheadFullException` / fallback while unrelated endpoints respond in `< 20ms`.

---

#### 16. Performance Testing

Integrating automated performance baselines into the testing pipeline (using **k6**, **Gatling**, or **JMeter**):
- Run lightweight **CI performance smoke gates** on pull requests/staging releases (e.g., 200 RPS for 5 min) verifying P95/P99 latency budgets and zero accidental N+1/memory regressions alongside full **Load, Stress, Soak, and Spike** suites before major releases.

---

#### 17. Testing Eventual Consistency

Never use `Thread.sleep(5000)` in asynchronous tests—it makes builds painfully slow on fast CI agents and still flakes randomly on loaded CI runners!

Always use **Awaitility** to poll deterministically until expected eventual state converges:

```java
// Trigger async order workflow
OrderResponse order = restClient.post().uri("/api/v1/orders")...;

// Poll cleanly until Kafka consumer / Outbox / CQRS read view converges!
await()
    .atMost(Duration.ofSeconds(10))
    .pollInterval(Duration.ofMillis(100))
    .untilAsserted(() -> {
        OrderSummaryView view = orderQueryRepository.findById(order.id()).orElseThrow();
        assertThat(view.getStatus()).isEqualTo("CONFIRMED");
    });
```

---

#### 18. Testing Saga Compensation

Testing *only* the Saga happy path leaves your most critical production recovery code untested. Write explicit tests for every **failure & compensation path**:

1. **Mid-Saga Failure → Reverse Compensation:**
   - Trigger `POST /orders`. Stub `PaymentService` (`201 Charged`) and stub `InventoryService` (`409 Out of Stock`).
   - Using **Awaitility**, assert:
     - `PaymentService` WireMock (or Kafka command topic) received the **`RefundPayment` compensation call**.
     - `Order` status in `OrderDB` transitioned `PENDING → CANCELLED`.
     - `Outbox` published `OrderCancelledEvent`.
2. **Saga Timeout + Late-Arriving Reply Race:**
   - Simulate `PaymentService` delaying past Saga step timeout → Order transitions to `CANCELLED` → then deliver a late `PaymentSucceededEvent` → assert Orchestrator immediately triggers automatic **`RefundPayment`**!
3. **Compensation Retry on Transient Failure:**
   - Stub `RefundPayment` to fail `503` twice, then `200 OK` → assert Orchestrator retries idempotently until compensation completes (and routes to compensation DLQ on permanent exhaustion).

---

#### 19. Testing Idempotency

Every mutating API endpoint requires **three automated idempotency test scenarios**:

1. **Sequential Replay (Same Key + Same Body):**
   - Call `POST /api/v1/payments` **twice** with the same `Idempotency-Key: K1` and identical JSON body.
   - Assert **both return `201 Created` with the exact same `paymentId`**, DB contains **strictly 1 row**, and downstream WireMock gateway received **strictly 1 HTTP call (`verify(1, ...)`)**.
2. **Payload Mismatch Guard (Same Key + Different Body):**
   - Call `POST /api/v1/payments` with `K1` (`amount: 100`), then again with `K1` (`amount: 500`) → assert **`422 Unprocessable Entity` / `409 Conflict`**!
3. **Concurrent Duplicate Race Condition (`CountDownLatch` Stress Test):**
   ```java
   int threads = 10;
   CountDownLatch ready = new CountDownLatch(1);
   CountDownLatch done = new CountDownLatch(threads);
   ExecutorService pool = Executors.newFixedThreadPool(threads);
   String key = UUID.randomUUID().toString();

   for (int i = 0; i < threads; i++) {
       pool.submit(() -> {
           try {
               ready.await(); // Release all 10 threads at the EXACT same millisecond!
               sendPaymentRequest(key, BigDecimal.valueOf(100));
           } finally { done.countDown(); }
       });
   }
   ready.countDown();
   done.await();

   assertThat(paymentRepository.count()).isEqualTo(1); // Strictly ONE payment created!
   ```

---

#### 20. Testing Duplicate Events

Verify consumer idempotency against **real Kafka redeliveries** and **out-of-order messages**:
1. **Sequential & Concurrent Duplicate Event Delivery:**
   - Produce two `ConsumerRecord`s to a Testcontainers Kafka topic with the **exact same `eventId`** (first directly, then again after 500ms—and also concurrently across partitions if testing shared key logic).
   - Assert via `Awaitility` that `inbox_processed_events` recorded `eventId` **once**, `inventory.available_qty` was decremented **only ONCE** (e.g., `10 → 8`, never `6`!), and **0 messages** landed in `.DLT`.
2. **Out-of-Order Event Guard:**
   - Publish `OrderUpdatedEvent(version = 3, status = "SHIPPED")` first, followed by stale `OrderUpdatedEvent(version = 2, status = "PAID")`.
   - Assert the CQRS projection stays at `status = "SHIPPED"` (`version = 3`) without regressing!

---

#### Quick Interview Revision Checklist (Section 10)
- Can you explain **why microservices shift from the classic Pyramid to the Testing Honeycomb** (maximizing Component + Contract tests, minimizing E2E)?
- Can you contrast **`@MockitoBean` in-process mocks vs WireMock HTTP stubs**—and explain why `@MockitoBean` hides timeout, serialization, and circuit-breaker bugs?
- Can you show how **Testcontainers (`@ServiceConnection` / Quarkus Dev Services)** solves H2-vs-PostgreSQL discrepancies, and why **`@Transactional` test rollback breaks `RANDOM_PORT` & async tests**?
- Can you walk through **Consumer-Driven Contract Testing (Pact / Spring Cloud Contract)** including Message Pact and `can-i-deploy`?
- Can you design automated tests for **Resilience4j (`Retry`, `CircuitBreaker`, `Bulkhead`, `Timeout`)**, **Eventual Consistency (`Awaitility`)**, **Saga Compensation**, **Concurrent Idempotency (`CountDownLatch`)**, and **Duplicate/Out-of-Order Kafka Events**?
- Can you distinguish **Failure Injection (`Toxiproxy`)** from production **Chaos Engineering** (Steady State → Hypothesis → Blast Radius → Abort Conditions)?
