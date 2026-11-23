# Microservices Study Notes — Section 13

## 13. Advanced Java Microservices Engineering

> *This section is particularly relevant to **IBM** and **Red Hat** because their roles involve Java-based backend development (Spring Boot, Quarkus, Jakarta EE / MicroProfile, and OpenShift).*

---

### 13.1 Application Internals

---

#### 1. Dependency Injection (DI)

**Dependency Injection** supplies an object's required collaborators externally rather than having the object instantiate (`new`) or look them up itself.

##### Constructor vs Field Injection
- **Constructor Injection (Recommended Standard):**
  - Allows `private final` immutable fields (safe publication across threads!).
  - Makes dependencies explicit, unit-testable with plain `new OrderService(mockRepo)` without booting Spring/CDI, and **catches circular dependencies (`A ↔ B`) immediately at startup**.
- **Field Injection (`@Autowired` / `@Inject` on private fields — Avoid!):**
  - Hides coupling (easy to accumulate 15 dependencies in one "God Service"), makes fields mutable, and forces reflection in tests.

##### Bean Scopes & The Self-Invocation Proxy Trap (Classic Interview Question!)
- **Scopes:** `@Singleton` / `@ApplicationScoped` (1 shared instance per container—must stay stateless!), `@RequestScoped` (1 per HTTP request), `@Prototype` / `@Dependent` (new instance per injection).
- **Proxy Self-Invocation Bug:** Cross-cutting annotations like **`@Transactional`**, **`@Async`**, **`@Cacheable`**, and **`@CircuitBreaker`** work via **AOP/CDI dynamic proxies** wrapping the bean!
  - Calling `this.saveOrder()` **from inside another method in the exact same class** invokes the raw `this` reference and **completely bypasses the proxy**—meaning `@Transactional` or `@CircuitBreaker` silently does **NOTHING!**
  - *Fix:* Extract the annotated method into a separate injected Spring/CDI bean (or inject self carefully in Quarkus/Spring only if unavoidable).

---

#### 2. Inversion of Control (IoC)

**Inversion of Control** means the framework container (Spring `ApplicationContext` or Quarkus **ArC** CDI container) controls object instantiation, dependency wiring, proxy wrapping, and lifecycle callbacks—instead of your `main()` method orchestrating `new` calls manually:
- **Spring Boot IoC:** Historically resolves bean graphs dynamically at **startup via classpath scanning & Java reflection** (plus optional Spring AOT build-time generation).
- **Quarkus ArC IoC:** Resolves the entire CDI dependency graph **100% at compile/build time via Jandex**, eliminates unused beans, and generates direct bytecode wiring (zero runtime reflection or classpath scanning!).

---

#### 3. Application Lifecycle

Understanding startup and shutdown hooks prevents race conditions and data loss:
1. **Startup Sequence:**
   - Config properties loaded → Bean definitions scanned/registered → Singleton beans instantiated & wired → `@PostConstruct` callbacks run → DB migrations (Flyway/Liquibase) execute → Embedded web server (Tomcat/Netty/Vert.x) binds port → `ApplicationReadyEvent` (Spring) / `@Observes StartupEvent` (Quarkus) → **Readiness Probe turns `UP`**.
2. **Graceful Shutdown Sequence (`SIGTERM`):**
   - `preStop` wait completes → Server stops accepting *new* HTTP requests (`server.shutdown=graceful`) → In-flight requests drain → Kafka consumers commit processed offsets & leave group cleanly → `@PreDestroy` / `@Observes ShutdownEvent` closes HikariCP/Agroal & Redis pools → JVM exits `0`.

---

#### 4. Request Lifecycle

```text
Spring MVC (Servlet / Tomcat Blocking Request Flow):
  HTTP ─► Tomcat Worker Thread ─► Servlet FilterChain (Security, OTel Trace, MDC)
      ─► DispatcherServlet ─► HandlerMapping ─► HandlerInterceptor.preHandle()
      ─► Jackson Deserialize + @Valid ─► @RestController ─► @Service (@Transactional)
      ─► Jackson Serialize ─► @RestControllerAdvice (on Exception) ─► postHandle / afterCompletion

Quarkus REST (Vert.x / Netty Non-Blocking or Virtual Thread Flow):
  HTTP ─► Vert.x Event Loop ─► @ServerRequestFilter ─► Router
      ─► (Event Loop if Uni/non-blocking | Worker/VirtualThread if blocking!)
      ─► Jakarta REST Resource Method ─► @ServerExceptionMapper ─► @ServerResponseFilter
```

---

#### 5. Filters and Interceptors

| Mechanism | Layer | Knows Target Method/Annotations? | Primary Use Cases |
| --- | --- | --- | --- |
| **Servlet `Filter`** (`OncePerRequestFilter`) | Servlet Container *(outside Spring MVC)* | No (sees raw `HttpServletRequest`/`Response`) | **CORS, Spring Security filter chain, JWT parsing, Gzip, Request/Response body caching, MDC/Trace wrapping** |
| **Spring `HandlerInterceptor`** | Spring MVC *(inside `DispatcherServlet`)* | **Yes** (`HandlerMethod` & custom annotations) | Endpoint-specific auth/audit checks, per-handler timing, tenant context binding |
| **Quarkus `@ServerRequestFilter` / `ContainerRequestFilter`** | Quarkus REST / Jakarta REST pipeline | Yes (`ResourceInfo`) | Non-blocking request/header inspection, early abort with `RestResponse`, security checks |
| **AOP / CDI Interceptor (`@AroundInvoke`, `@Aspect`)** | Any Spring/CDI Bean method | **Yes** (`MethodInvocation` / `ProceedingJoinPoint`) | **`@Transactional`**, Resilience4j **`@CircuitBreaker`**, custom `@AuditLog` |

---

#### 6. Exception Handling

Centralize exception-to-HTTP mapping cleanly instead of scattering `try/catch` blocks inside every controller:
- **Spring Boot 3:** `@RestControllerAdvice` + `@ExceptionHandler(OrderNotFoundException.class)` returning **`ProblemDetail` (RFC 9457)**.
- **Quarkus:** `@ServerExceptionMapper` (or Jakarta REST `ExceptionMapper<T>`) returning `RestResponse<ProblemDetail>` (non-blocking compatible!).
- **Golden Rules:** Distinguish **4xx Domain Exceptions** (validation, conflict, missing entity—log at `INFO`/`WARN`) from **5xx Unexpected Errors** (log full stack trace at `ERROR` with `traceId`, return generic safe external message!).

---

#### 7. Validation

Enforce **Jakarta Bean Validation (`jakarta.validation`)** at the API boundary:
- Annotate Request DTO Java `record`s (`@NotBlank`, `@Positive`, `@Size(max = 100)`, `@Email`, `@Pattern`, `@Valid` nested collections) and trigger via `@Valid` in controller parameters.
- Use `@Validated` on `@Service` classes for method-level contract validation (`ConstraintViolationException`).
- Write custom domain annotations (`@ValidCurrency`, `@ValidDateRange`) implementing `ConstraintValidator<A, T>`.

---

#### 8. Serialization

Best practices for **Jackson (`ObjectMapper`)** in high-throughput Java microservices:
- Map requests/responses to immutable **Java `record` DTOs** (Jackson supports `record`s natively without no-arg constructors!).
- Always configure **`DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES = false`** (or `@JsonIgnoreProperties(ignoreUnknown = true)`) for forward-compatible schema evolution.
- Disable **`WRITE_DATES_AS_TIMESTAMPS`** so `Instant` / `OffsetDateTime` serialize cleanly as ISO-8601 UTC strings (`"2026-11-23T10:30:00Z"`).
- **Native Image Gotcha:** Reflection-based serialization fails in GraalVM Native unless DTOs are annotated with `@RegisterForReflection` or scanned at build time (`quarkus-rest-jackson` handles REST DTOs automatically at build time!).

---

#### 9. Configuration Management

Externalized configuration precedence in Spring Boot & Quarkus (highest wins):
1. Command-line arguments (`--server.port=9090`)
2. OS Environment Variables (`SPRING_DATASOURCE_PASSWORD`, `QUARKUS_HTTP_PORT`)
3. External config files / Kubernetes ConfigMaps & Vault mounted files (`spring.config.import=configtree:/etc/secrets/`)
4. Profile-specific YAML/properties (`application-prod.yml`)
5. Packaged default `application.yml` / `application.properties` inside the JAR

##### Type-Safe Immutable Binding
- **Spring Boot:** `@ConfigurationProperties(prefix = "clients.payment")` + `@Validated` on a Java **`record`** (validated at startup—pod fails fast on boot if a required property is missing!).
- **Quarkus (SmallRye Config):** `@ConfigMapping(prefix = "clients.payment")` on a Java **`interface`**.

---

#### 10. Environment-Specific Configuration

- **Spring Profiles:** Create `application-dev.yml`, `application-prod.yml` and activate via `SPRING_PROFILES_ACTIVE=prod` (or `@Profile("dev")` for dev-only test beans).
- **Quarkus Profiles:** Prefix properties directly inside one `application.properties` (`%dev.quarkus.datasource...`, `%test...`, `%prod...`)—activated via `QUARKUS_PROFILE=prod`.
- **Golden Rule:** Never compile separate artifacts per environment (`order-prod.jar` vs `order-qa.jar`); promote the **exact same immutable container image** across environments with injected config!

---

### 13.2 Concurrency

---

#### 1. Threads and Thread Pools

| Thread Model | Stack & Scheduling | Best Use Case | Key Production Gotcha |
| --- | --- | --- | --- |
| **Platform OS Threads** | ~1MB native stack each; scheduled by OS kernel (limit ~few hundred–few thousand) | **CPU-bound** work, legacy libraries | High RAM & context-switch overhead under thousands of blocked I/O calls |
| **Java 21+ Virtual Threads** (`Project Loom`) | Lightweight heap-allocated stack chunks; scheduled by JVM onto small ForkJoin carrier pool | **Blocking I/O workloads** (REST calls, JDBC, Kafka) at massive concurrency! | **NEVER pool Virtual Threads** (create 1 per task); on **JDK 21**, avoid blocking I/O inside `synchronized` blocks (**causes carrier thread pinning**—use `ReentrantLock`! *Fixed in JDK 24 JEP 491*) |

---

#### 2. ExecutorService

Never create threads with raw `new Thread(...)` or unsafe factory defaults:
- **Avoid `Executors.newFixedThreadPool()`:** Uses an **unbounded `LinkedBlockingQueue`**, which silently buffers millions of tasks during slowdowns until **`OutOfMemoryError`**!
- **Avoid `Executors.newCachedThreadPool()`:** Spawns **unlimited OS platform threads** under bursts until OS crashes (`unable to create new native thread`).
- **Production `ThreadPoolExecutor` Pattern:**
  - Always set explicit `corePoolSize`, `maxPoolSize`, **bounded `ArrayBlockingQueue`**, custom `ThreadFactory` (meaningful thread names!), and an explicit `RejectedExecutionHandler` (`AbortPolicy` to fail fast, or `CallerRunsPolicy` for gentle backpressure).
  - Or for blocking I/O on Java 21+: **`Executors.newVirtualThreadPerTaskExecutor()`**!
  - On shutdown: call `executor.shutdown()`, await `awaitTermination(30, SECONDS)`, then `shutdownNow()`.

---

#### 3. CompletableFuture

Ideal for **parallel Scatter–Gather aggregation** across independent microservice calls:

```java
// NEVER omit the custom Executor on blocking I/O (otherwise blocks ForkJoinPool.commonPool!)
CompletableFuture<Customer> customerFuture = CompletableFuture
    .supplyAsync(() -> customerClient.getById(id), ioExecutor)
    .orTimeout(800, TimeUnit.MILLISECONDS)
    .exceptionally(ex -> Customer.unknown());

CompletableFuture<List<Order>> ordersFuture = CompletableFuture
    .supplyAsync(() -> orderClient.getByCustomer(id), ioExecutor)
    .orTimeout(800, TimeUnit.MILLISECONDS);

DashboardDto dashboard = customerFuture
    .thenCombine(ordersFuture, DashboardDto::new)
    .join();
```
- Key combinators: `thenApply` (sync map), `thenCompose` (flatMap async future), `thenCombine` (join 2 parallel futures), `allOf` / `anyOf`, `orTimeout` / `completeOnTimeout` (JDK 9+).

---

#### 4. Synchronization

- **`synchronized`:** Built-in intrinsic monitor lock providing **mutual exclusion + Java Memory Model (JMM) happens-before memory visibility**. Keep synchronized sections tiny (pure fast in-memory updates, never network/DB I/O inside `synchronized`!).
- **`volatile`:** Guarantees **cross-thread visibility and instruction-reordering protection** directly via main memory barriers—**does NOT guarantee atomicity**! `volatile int count; count++;` is **still a race condition** (read + increment + write = 3 ops; use `AtomicInteger` / `LongAdder` instead!).

---

#### 5. Locks

`java.util.concurrent.locks`:
- **`ReentrantLock`:** Supports **`tryLock(timeout, unit)`** (prevents indefinite stalls!), `lockInterruptibly()`, multiple `Condition` queues, and **does NOT pin Java 21 Virtual Threads**! Always unlock inside `try { ... } finally { lock.unlock(); }`.
- **`ReentrantReadWriteLock` / `StampedLock`:** Allows unlimited concurrent readers while giving writers exclusive locks (`StampedLock` even offers lock-free optimistic reads!).
- **Coordinators:** `Semaphore` (concurrency bulkhead), `CountDownLatch` (one-shot barrier—great for concurrent race testing), `CyclicBarrier`.

---

#### 6. Deadlocks

A **Deadlock** permanently freezes two or more threads waiting circularly for each other's locks (**4 Coffman Conditions:** Mutual Exclusion, Hold and Wait, No Preemption, Circular Wait):
```text
Thread 1: synchronized(accountA) { synchronized(accountB) { ... } }
Thread 2: synchronized(accountB) { synchronized(accountA) { ... } }  ──► DEADLOCK!
```
- **Prevention:**
  1. **Strict Global Lock Ordering:** Always acquire locks in deterministic sorted ID order (`if (fromId.compareTo(toId) < 0) lock(from) then lock(to)`).
  2. **Timed `tryLock(500, MILLISECONDS)`:** Back off and release first lock if second lock times out.
  3. Never invoke external callbacks/HTTP calls while holding a lock.
- **Detection:** `jcmd <pid> Thread.print` or `jstack <pid>` detects and prints Java deadlocks immediately!

---

#### 7. Race Conditions

Two classic bugs in concurrent Java services:
1. **Check-Then-Act:**
   ```java
   // BUG: Two threads both see key missing and overwrite each other!
   if (!cache.containsKey(id)) { cache.put(id, load(id)); }
   // FIX: Atomic Map operation (or DB INSERT ... ON CONFLICT DO NOTHING)
   cache.computeIfAbsent(id, this::load);
   ```
2. **Read-Modify-Write:** `counter++` loses increments → fix with **`AtomicLong`** / **`LongAdder`** (uses hardware CAS `Compare-And-Swap`, with `LongAdder` stripping contention across cells under heavy concurrency!).

---

#### 8. Thread Safety

Build thread-safe microservices using this **4-step priority hierarchy**:
1. **Statelessness (Best):** Never store request-specific mutable fields inside `@Service` or `@RestController` singletons!
2. **Immutability:** Use Java **`record`s**, `final` fields, and `List.copyOf(...)` / `Map.copyOf(...)`—immutable objects are automatically 100% thread-safe with zero synchronization!
3. **Thread-Safe Primitives:** `ConcurrentHashMap`, `AtomicReference`, `LongAdder`.
4. **Explicit Bounded Locking (`ReentrantLock`)** only when multi-field atomic invariants require it.

---

#### 9. Concurrent Collections

- **`ConcurrentHashMap`:** Thread-safe high-concurrency map using CAS on empty buckets + fine-grained per-bucket head node locking; lock-free `get()`; weakly consistent fail-safe iterators (**never allows `null` keys or values!**). Replaces legacy `Hashtable` and `Collections.synchronizedMap`.
- **`CopyOnWriteArrayList`:** Copies the entire underlying array on every `add`/`set`; zero locking on reads/iteration—ideal **only** for tiny, read-99.9%-of-the-time lists (e.g., event listeners).
- **`ArrayBlockingQueue` (bounded!) / `ConcurrentLinkedQueue`:** Producer-consumer pipelines.

---

#### 10. Thread Starvation

Occurs when runnable tasks cannot make progress because worker threads are all occupied:
- **Classic Causes in Java Microservices:**
  1. **Missing HTTP/DB timeouts** on Tomcat worker pool.
  2. **Blocking the Event Loop (Netty / Vert.x):** Calling `jdbc`, `RestTemplate`, or `Thread.sleep` directly on one of the 8 Netty/Vert.x I/O event-loop threads freezes thousands of multiplexed connections at once! (*Detect automatically in tests using **BlockHound** or Quarkus's built-in event-loop blocking detector!*)
  3. **Virtual Thread Pinning (JDK 21):** Long network/DB wait inside a `synchronized` block pins the underlying OS carrier thread.

---

#### 11. Context Propagation

Because `SLF4J MDC` (`traceId`), Spring `SecurityContextHolder`, and OpenTelemetry `Context` live inside **`ThreadLocal`** variables, spawning a `CompletableFuture`, `@Async`, Thread-Pool Bulkhead, or Reactor `Mono` **drops all tracing, logging, and user security context on the worker thread!**

- **Solutions:**
  - **Spring Boot 3 / Micrometer Context Propagation:** Wrap executors with **`ContextExecutorService.wrap(executor)`** / `ContextSnapshotFactory` (or `Hooks.enableAutomaticContextPropagation()` in WebFlux) and Spring `TaskDecorator` for `@Async`.
  - **Quarkus:** **`quarkus-smallrye-context-propagation`** automatically propagates CDI, Security, RESTEasy/OTel contexts across Mutiny & worker threads.
  - **Java `ScopedValue` (Modern Replacement for `ThreadLocal`):** Immutable, bounded-lifetime context sharing designed specifically for massive numbers of Virtual Threads (`ScopedValue.where(USER_ID, "u-42").run(...)`).

---

#### 12. Blocking versus Non-Blocking Execution

| Dimension | Blocking (Tomcat + Platform Threads) | Blocking + **Java 21 Virtual Threads** | Non-Blocking Reactive (Netty / Vert.x + Reactor / Mutiny) |
| --- | --- | --- | --- |
| **Concurrency Ceiling** | Limited (~200–1,000 concurrent blocking calls) | **High (10,000–100,000+ concurrent I/O calls)** | **Highest** with minimal RAM/threads (~cores × 2 event-loop threads) |
| **Programming Style** | Simple imperative | **Simple imperative** | Functional callbacks (`Mono`/`Flux`, `Uni`/`Multi`) |
| **Stack Traces & Debugging** | Clean readable stack traces | Clean readable stack traces | Harder async stack traces |
| **Ecosystem / JDBC** | Works with all standard JDBC/JPA libs | Works with standard JDBC/JPA (watch `synchronized` pinning on JDK 21) | Requires 100% reactive drivers (**R2DBC / Reactive Postgres Client**—one blocking call stalls the event loop!) |

---

### 13.3 Spring Boot and Quarkus Implementation

---

#### 1. REST Controllers and Resource Classes

| Spring Boot 3 (Spring MVC / WebFlux) | Quarkus (Quarkus REST / Jakarta REST) |
| --- | --- |
| `@RestController` + `@RequestMapping("/api/v1/orders")` | `@Path("/api/v1/orders")` + `@Produces/@Consumes(MediaType.APPLICATION_JSON)` |
| `@GetMapping("/{id}")`, `@PostMapping` | `@GET`, `@Path("/{id}")`, `@POST` |
| Return `ResponseEntity<OrderDto>` | Return `RestResponse<OrderDto>` or direct DTO (`200 OK` / `201 Created`) |

```java
// Spring Boot 3
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService service;
    public OrderController(OrderService service) { this.service = service; }

    @PostMapping
    public ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req) {
        OrderResponse created = service.placeOrder(req);
        return ResponseEntity.created(URI.create("/api/v1/orders/" + created.id())).body(created);
    }
}

// Quarkus Equivalent
@Path("/api/v1/orders")
public class OrderResource {
    private final OrderService service;
    public OrderResource(OrderService service) { this.service = service; }

    @POST
    public RestResponse<OrderResponse> create(@Valid CreateOrderRequest req) {
        OrderResponse created = service.placeOrder(req);
        return RestResponse.created(URI.create("/api/v1/orders/" + created.id()));
    }
}
```

---

#### 2. Service and Repository Layers

- **Spring Boot:** `@Service` + `@Transactional` wrapping **Spring Data JPA** (`interface OrderRepository extends JpaRepository<Order, UUID>`) or `JdbcClient`.
- **Quarkus:** `@ApplicationScoped` + `@Transactional` wrapping **Hibernate ORM with Panache**:
  - *Repository Pattern:* `implements PanacheRepository<Order>` (clean DDD separation, easiest to mock in tests!).
  - *Active Record Pattern:* `Order extends PanacheEntity` (`Order.findById(id)`, `order.persist()`).

---

#### 3. Dependency Injection (Spring vs Quarkus ArC)

- **Spring Boot:** Uses `@Service`, `@Component`, `@Repository`, `@Configuration`/`@Bean` resolved dynamically via Spring `ApplicationContext` (supports constructor injection seamlessly even without `@Autowired` on single-constructor classes!).
- **Quarkus ArC (Build-Time CDI):** Uses Jakarta CDI (`@ApplicationScoped`, `@Singleton`, `@Inject`, constructor injection without needing `@Inject` on single constructor). **ArC resolves 100% of bean wiring during `mvn package`**, removes dead/unused beans automatically, and avoids reflection proxies wherever possible!

---

#### 4. Configuration Properties

```java
// Spring Boot: Immutable record validated at startup
@Validated
@ConfigurationProperties(prefix = "clients.payment")
public record PaymentClientProps(@NotBlank String baseUrl, @Positive int timeoutMs) {}

// Quarkus (SmallRye Config): Type-safe interface
@ConfigMapping(prefix = "clients.payment")
public interface PaymentClientConfig {
    String baseUrl();
    int timeoutMs();
}
```

---

#### 5. REST Clients

- **Spring Boot 3:**
  - **`RestClient`** (modern fluent imperative client—scales cleanly with Virtual Threads).
  - **`@HttpExchange` (Spring HTTP Interface):** Declarative interface backed by either `RestClient` (blocking) or `WebClient` (reactive).
  - **Spring Cloud OpenFeign** (legacy/maintenance declarative client).
- **Quarkus (`quarkus-rest-client`):**
  - Declarative `@RegisterRestClient(configKey = "payment-api")` interface built atop the non-blocking **Vert.x HTTP client**—supports both blocking return types and reactive `Uni<T>` return types seamlessly!

---

#### 6. Exception Handling

- **Spring Boot 3:** `@RestControllerAdvice` + `@ExceptionHandler(DomainException.class)` returning native **`ProblemDetail` (RFC 9457)** (`spring.mvc.problemdetails.enabled=true`).
- **Quarkus:** `@ServerExceptionMapper` methods (e.g., `public RestResponse<Problem> mapNotFound(OrderNotFoundException ex)`)—registered without reflection at build time!

---

#### 7. Validation

Both use **Hibernate Validator** implementing **Jakarta Bean Validation (`jakarta.validation`)**:
- Spring Boot: `spring-boot-starter-validation`
- Quarkus: `quarkus-hibernate-validator`
- Identical `@Valid`, `@NotNull`, `@NotBlank`, `@Positive`, `@Pattern` annotations on Java `record` DTOs.

---

#### 8. Health Checks

| Probe / Feature | Spring Boot Actuator (`spring-boot-starter-actuator`) | Quarkus SmallRye Health (`quarkus-smallrye-health`) |
| --- | --- | --- |
| **Liveness Endpoint** | `/actuator/health/liveness` | `/q/health/live` (`@Liveness` CDI bean) |
| **Readiness Endpoint** | `/actuator/health/readiness` | `/q/health/ready` (`@Readiness` CDI bean) |
| **Startup Endpoint** | `/actuator/health/startup` | `/q/health/started` (`@Startup` CDI bean) |
| **K8s Probe Activation** | `management.endpoint.health.probes.enabled=true` | Enabled automatically when `quarkus-kubernetes` or `quarkus-smallrye-health` is added! |

---

#### 9. Metrics

Both Spring Boot and Quarkus standardize on **Micrometer** for Prometheus integration:
- **Spring Boot:** `micrometer-registry-prometheus` exposes `/actuator/prometheus`.
- **Quarkus:** `quarkus-micrometer-registry-prometheus` exposes `/q/metrics` (with zero-reflection build-time metered annotations `@Timed`, `@Counted`).

---

#### 10. Distributed Tracing

- **Spring Boot 3:** `micrometer-tracing-bridge-otel` + `opentelemetry-exporter-otlp` exports W3C traces (`traceparent`) over OTLP gRPC/HTTP (or attach the OpenTelemetry Java Agent).
- **Quarkus:** `quarkus-opentelemetry` integrates OpenTelemetry natively at build time (propagating W3C `traceparent` across Vert.x HTTP, REST Client, Hibernate/JDBC, SmallRye Reactive Messaging/Kafka, and compiling cleanly into **GraalVM Native Images**!).

---

#### 11. Resilience4j (and SmallRye Fault Tolerance)

- **Spring Boot:** Uses **Resilience4j** (`resilience4j-spring-boot3`) providing `@CircuitBreaker`, `@Retry`, `@Bulkhead`, `@TimeLimiter`, `@RateLimiter` configured in `application.yml`.
- **Quarkus (Red Hat Standard):** Uses **SmallRye Fault Tolerance** (`quarkus-smallrye-fault-tolerance`, implementing **Eclipse MicroProfile Fault Tolerance**): `@CircuitBreaker`, `@Retry`, `@Bulkhead`, `@Timeout`, `@RateLimit`, and `@Fallback`—compatible with both imperative and reactive `Uni<T>` methods!

---

#### 12. Reactive Programming

| Feature | Spring WebFlux (**Project Reactor**) | Quarkus (**SmallRye Mutiny** over Vert.x) |
| --- | --- | --- |
| **0..1 Async Result** | `Mono<T>` | `Uni<T>` |
| **0..N Async Stream** | `Flux<T>` | `Multi<T>` |
| **API Ergonomics** | Dense functional operators (`flatMap`, `zipWhen`, `onErrorResume`) | Explicit event-driven fluency (`onItem().transform(...)`, `onFailure().recoverWithItem(...)`) |
| **Engine** | Project Reactor + Netty | **Eclipse Vert.x** (single unified reactive core) |

---

#### 13. Blocking versus Non-Blocking I/O

- **Spring Boot:** You generally choose *one* web stack per microservice: **Spring MVC** (Tomcat blocking / Virtual Threads) OR **Spring WebFlux** (Netty non-blocking + R2DBC; mixing JPA/Tomcat libraries causes blocking hazards).
- **Quarkus Unified Reactive + Imperative Engine:** Everything runs on **Eclipse Vert.x** at the core! On a single microservice:
  - Methods returning `Uni<T>` run non-blocking directly on the **Vert.x Event Loop**.
  - Traditional blocking methods (standard Hibernate ORM/JDBC) automatically dispatch to **Worker Threads**.
  - Methods annotated with **`@RunOnVirtualThread`** run on **Java 21+ Virtual Threads**—giving non-blocking scalability with simple imperative code!

---

#### 14. Quarkus Build-Time Optimizations

*(Core architectural differentiator frequently asked in **Red Hat / IBM** interviews!)*

Traditional Java frameworks do heavy metadata work **at runtime on every single container boot**:
1. Scan JARs/classpath for `@Entity`, `@Service`, `@RestController` annotations
2. Parse YAML/XML configuration files
3. Build reflection metadata, Hibernate ORM metamodels, and CGLIB dynamic proxies
4. Initialize beans and frameworks—wasting hundreds of MBs of heap/Metaspace and `5–20 seconds` per pod startup!

**How Quarkus Optimizes via Build-Time Augmentation (`mvn package`):**
- **Build Time (`BUILD_TIME` phase):** Uses **Jandex** bytecode indexer to read annotations once, resolves the entire **ArC** CDI graph, compiles routes, builds Hibernate ORM metamodels, generates reflection-free bytecode proxies, strips dead/unused initialization code, and records pre-initialized bytecode steps.
- **Runtime (`STATIC_INIT` + `RUNTIME_INIT`):** Executes only the minimal pre-computed bytecode steps—**zero classpath scanning, minimal reflection, ~50%–80% lower memory footprint, and sub-second JVM startup (~0.5–1.5s)!**

---

#### 15. Native Image Considerations

Compiling Java Ahead-of-Time (**AOT**) into a standalone OS machine-code executable using **GraalVM Native Image** (or **Red Hat Mandrel**, Red Hat's downstream OpenJDK/GraalVM distribution tailored for Quarkus):

##### Closed-World Assumption & Constraints
Because Native Image performs static whole-program reachability analysis at build time, **all reachable classes and behaviors must be discoverable at compile time**:
1. **Dynamic Reflection / Proxies / Resource Loading:** Fails silently at runtime (`ClassNotFoundException`) unless explicitly registered (`@RegisterForReflection`, `reflect-config.json`—automated cleanly by Quarkus extensions and Spring Boot AOT hints).
2. **Dynamic Classloading:** No loading arbitrary runtime `.jar` plugins or dynamic bytecode generation at runtime.
3. **Long, Heavy CI Builds:** Native compilation takes **1–4 minutes** and consumes **4–8 GB RAM** per build container (keep running regular tests on JVM; run Native compile in CI release pipeline!).
4. **Different GC & Tooling:** Community GraalVM/Mandrel Native Image uses **SerialGC** (or G1 in Oracle GraalVM Enterprise) instead of HotSpot's Generational ZGC/Shenandoah, and traditional HotSpot tools (`jmap`, dynamic `-javaagent` OTel agents) do not work on native binaries (must compile OTel into the binary via Quarkus/Spring build extensions!).

---

#### 16. Spring Boot versus Quarkus

| Dimension | Spring Boot 3 | Quarkus (Red Hat) |
| --- | --- | --- |
| **Ecosystem & Talent Pool** | Massive community, richest third-party starter ecosystem | Fast-growing cloud-native ecosystem; Jakarta EE & MicroProfile standards |
| **IoC & Boot Architecture** | Runtime reflection/DI + optional Spring AOT | **Build-time augmentation first** (Jandex + ArC CDI) |
| **Startup Speed (JVM Mode)** | ~2.5–8 seconds | **~0.5–1.5 seconds** |
| **Memory Footprint (JVM RSS)** | Higher (~300–700MB) | **Lower (~120–300MB)** |
| **Developer Experience** | Spring DevTools, Docker Compose / Testcontainers `@ServiceConnection` | **Quarkus Dev Mode (`quarkus dev`)**: instant live reload (<1s) + **zero-config Dev Services** (auto-starts Postgres/Kafka/Keycloak) |
| **Reactive Stack** | Separate WebFlux (Reactor/Netty) vs MVC (Tomcat) | Unified **Vert.x + Mutiny (`Uni`/`Multi`) + `@RunOnVirtualThread`** |
| **Native Image Support** | Spring Boot 3 AOT (good, evolving third-party hints) | **First-class native from day 1** (built alongside Red Hat Mandrel) |
| **Best Fit** | Large enterprise Spring teams, existing Spring Cloud estates, high-throughput long-lived JVM services | **Kubernetes/OpenShift-native**, dense multi-service clusters, serverless/Knative scale-to-zero, memory-constrained microservices |

---

#### 17. JVM versus Native Execution

| Factor | HotSpot JVM (JIT — G1GC / ZGC) | Native Image (AOT — GraalVM / Mandrel) |
| --- | --- | --- |
| **Startup Time** | Slower (`1–8s` + JIT warm-up period) | **Instant (`20–80ms`, ready immediately!)** |
| **Memory (RSS)** | Higher (needs RAM for JVM, Metaspace, JIT code cache, heap) | **Ultra-low (`30–100MB` — fits 3x–5x more pods per K8s node!)** |
| **Peak Throughput & P99 (Long-Running)** | **Higher!** JIT C2 profile-guided optimization + **Generational ZGC/Shenandoah** beat AOT after warm-up! | Slightly lower peak throughput (no runtime speculative JIT inlining, unless PGO is used) |
| **Container Image Size** | ~150–250MB (App + JRE) | ~50–100MB (UBI Micro / Distroless + binary) |
| **Build Time & Observability** | Fast builds (~10–20s); full JFR / `async-profiler` / `jstack` / Java Agents | Slow heavy builds (1–4 min); limited runtime JVM diagnostics |
| **When to Choose** | **Long-running, high-throughput stateful/OLTP services** where peak TPS & low P99 GC matter more than cold-start | **Serverless / Knative scale-to-zero, short-lived K8s Jobs/CLIs, rapid autoscaling spikes, & tight memory density** |

---

#### Quick Interview Revision Checklist (Section 13)
- Can you explain **Constructor Injection** and why **self-invocation (`this.method()`) bypasses Spring `@Transactional` / Quarkus `@CircuitBreaker` proxies**?
- Can you trace the full **Spring MVC Request Lifecycle** vs **Quarkus Vert.x Event Loop** and contrast **`Filter` vs `HandlerInterceptor` vs CDI Interceptor**?
- Can you explain **Java 21 Virtual Threads** (why **never pool** them, **carrier thread pinning** with `synchronized` vs `ReentrantLock`, and `ScopedValue` vs `ThreadLocal`)?
- Can you explain why **`Executors.newFixedThreadPool` / `ForkJoinPool.commonPool()`** are dangerous for blocking microservice calls, and how **Context Propagation (`MDC`/`SecurityContext`/OTel)** works across threads?
- Can you contrast **`volatile` vs `synchronized` vs `AtomicLong`/`LongAdder`**, explain **4 Coffman deadlock conditions**, and compare **`ConcurrentHashMap` vs `CopyOnWriteArrayList`**?
- For **Red Hat / IBM**, can you clearly explain **Quarkus build-time augmentation (Jandex + ArC)**, **Mutiny (`Uni`/`Multi`) vs Reactor (`Mono`/`Flux`)**, **Mandrel Native Image closed-world constraints**, and **when HotSpot JVM JIT beats Native Image**?
