# Microservices Study Notes — Section 2

## 2. Inter-Service Communication

> **Very high priority**

---

### 2.1 Synchronous Communication

---

#### 1. REST API Communication

In **synchronous communication**, the calling microservice sends a request over a network and **waits (blocks or awaits a future/reactive signal)** until the target service returns a response or times out.

**REST (Representational State Transfer)** over HTTP/HTTPS is the most common synchronous communication style between microservices and external clients because:
- Every language, framework, proxy, API gateway, and browser natively understands HTTP and JSON.
- Tools like `curl`, Postman, Swagger UI, and API gateways inspect HTTP methods, paths, and status codes out of the box.
- **Best suited for:** Real-time queries (`GET /orders/123`), immediate user-facing validations (`POST /auth/token`), and simple request-reply commands where the caller needs an immediate result.

---

#### 2. HTTP Fundamentals

HTTP is a stateless, text-based (HTTP/1.1) or binary-framed (HTTP/2) application-layer protocol built on top of TCP/TLS.

##### Anatomy of an HTTP Request & Response
```text
REQUEST:
  POST /api/v1/orders HTTP/1.1                 ◄── Request Line (Method, Path, Version)
  Host: order-service.prod.svc.cluster.local   ◄── Headers
  Content-Type: application/json
  Authorization: Bearer eyJhbGci...
  Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd
                                               ◄── Empty line (CRLF)
  {"customerId":"c-101","total":49.99}         ◄── Optional Body

RESPONSE:
  HTTP/1.1 201 Created                         ◄── Status Line (Version, Status Code, Reason)
  Content-Type: application/json
  Location: /api/v1/orders/ord-778             ◄── Response Headers
                                               ◄── Empty line
  {"orderId":"ord-778","status":"CREATED"}     ◄── Response Body
```

- **Stateless by protocol:** Each HTTP request contains all authentication, routing, and payload context needed to process it; servers do not store client session state in memory across requests.
- **URI Structure:** `scheme://host:port/path?query#fragment` (note: query strings filter/sort resources; paths identify resources).

---

#### 3. REST Architectural Constraints

Roy Fielding defined **6 architectural constraints** for true RESTful systems:

| Constraint | Meaning in Microservices |
| --- | --- |
| **1. Client–Server** | Separation of concerns: client handles UI/presentation; server owns data storage and domain rules. |
| **2. Stateless** | Server retains no client session state between requests. Auth/state travels in tokens (`Authorization`), headers, or DB. Enables horizontal scaling behind load balancers. |
| **3. Cacheable** | Responses explicitly state cacheability (`Cache-Control`, `ETag`, `Last-Modified`) so clients/CDNs/gateways can skip redundant calls. |
| **4. Uniform Interface** | Standardized interaction via: (a) Resource identification in URIs, (b) Manipulation through representations (JSON), (c) Self-descriptive messages (headers + status codes), (d) HATEOAS (hypermedia links in responses). |
| **5. Layered System** | Client doesn't know if it talks directly to the origin service or through an API Gateway, load balancer, or CDN. |
| **6. Code on Demand** *(optional)* | Server returns executable code (e.g., JavaScript to browsers); rarely used in service-to-service APIs. |

##### Richardson Maturity Model
- **Level 0 (Swamp of POX):** Single URI (`/api`), everything is `POST` (RPC-style, SOAP).
- **Level 1 (Resources):** Distinct resource URIs (`/orders`, `/orders/123`), still mostly `POST`.
- **Level 2 (HTTP Verbs & Status Codes):** Proper `GET`/`POST`/`PUT`/`PATCH`/`DELETE` + meaningful HTTP status codes (**industry standard for microservices**).
- **Level 3 (HATEOAS):** Responses embed hypermedia `_links` telling clients valid next state transitions.

---

#### 4. HTTP Methods and Status Codes

##### HTTP Methods Comparison
- **Safe:** Does not mutate server state (read-only).
- **Idempotent:** Calling it 1 time or $N$ times produces the **same server state** (critical for safe network retries).

| Method | Purpose | Safe? | Idempotent? | Request Body? |
| --- | --- | --- | --- | --- |
| **`GET`** | Retrieve resource representation | Yes | Yes | No |
| **`HEAD`** | Same as `GET`, headers only (no body) | Yes | Yes | No |
| **`OPTIONS`** | Discover supported methods / CORS preflight | Yes | Yes | No |
| **`PUT`** | Full replacement (or upsert) of resource at URI | No | Yes | Yes |
| **`DELETE`** | Delete resource at URI | No | Yes | Optional |
| **`POST`** | Create subordinate resource or trigger non-idempotent action | No | **No** *(unless `Idempotency-Key` used)* | Yes |
| **`PATCH`** | Partial update of resource | No | **Not guaranteed** *(idempotent if setting absolute values, not if incrementing `balance += 10`)* | Yes |

##### Essential HTTP Status Codes in Microservices

| Code | Meaning | Microservice Usage & Troubleshooting Context |
| --- | --- | --- |
| **`200 OK`** | Standard success | `GET`, `PUT`, `PATCH`, or synchronous `POST` action returning body |
| **`201 Created`** | New resource created | Return `Location: /orders/{id}` header + created DTO |
| **`202 Accepted`** | Accepted for **async processing** | Long-running job queued to Kafka/worker; return status polling URL |
| **`204 No Content`** | Success with empty response body | `DELETE` or `PUT` updates needing no response body |
| **`304 Not Modified`** | Client cache is still valid | Conditional `GET` using `If-None-Match` (`ETag`) |
| **`400 Bad Request`** | Malformed syntax / JSON / validation error | Missing required fields, invalid JSON format |
| **`401 Unauthorized`** | **Unauthenticated** (missing/expired/invalid JWT) | Client needs to authenticate / refresh token |
| **`403 Forbidden`** | **Authenticated, but lacks permission/role** | Valid JWT, but missing required scope/RBAC role |
| **`404 Not Found`** | Resource or route doesn't exist | Missing entity ID or wrong API Gateway route |
| **`409 Conflict`** | State or concurrency conflict | Optimistic locking version mismatch (`@Version`), duplicate unique key |
| **`422 Unprocessable Entity`** | Syntactically valid JSON, violates business rule | E.g., refund amount exceeds captured amount |
| **`429 Too Many Requests`** | Rate limit / quota exceeded | Include `Retry-After` header; client should back off |
| **`500 Internal Server Error`** | Unhandled exception / bug on server | Never leak stack traces in response body |
| **`502 Bad Gateway`** | Proxy/Gateway got invalid/reset response from upstream | Downstream pod crashed mid-response or malformed HTTP |
| **`503 Service Unavailable`** | Server overloaded, starting up, or circuit open | Pod failing readiness probe, bulkhead full, maintenance |
| **`504 Gateway Timeout`** | Gateway/proxy timed out waiting for upstream response | Downstream slow DB query / hung thread; **never blindly retry non-idempotent POST** |

---

#### 5. Request-Response Lifecycle

Every synchronous microservice call traverses multiple layers on both caller and callee:

```text
Caller Service (OrderService)
  1. Serialize DTO ──► JSON bytes (Jackson)
  2. Lease connection from HTTP Connection Pool
  3. (If cold) DNS lookup ─► TCP 3-way handshake ─► TLS handshake
  4. Write HTTP request + trace headers (traceparent, Authorization)
        │
        ▼  [Network / API Gateway / Load Balancer / Sidecar Proxy]
        │
Callee Service (PaymentService)
  5. OS Kernel TCP buffer ─► Web Server (Tomcat / Netty / Undertow)
  6. Filter/Interceptor Chain (Logging, Auth/JWT verification, Metrics, Trace Span)
  7. Dispatcher / Router ─► Controller method + Bean Validation (@Valid)
  8. Service Layer ─► Domain logic + DB connection pool query
  9. Serialize response DTO ─► JSON bytes ─► Write HTTP status + body
        │
        ▼
Caller Service
 10. Read response bytes ─► Deserialize JSON ─► Return connection to pool (keep-alive)
```

---

#### 6. HTTP Headers and Content Negotiation

##### Critical Inter-Service Headers
- **`Content-Type`:** Format of the request/response body (`application/json`, `application/problem+json`, `application/protobuf`).
- **`Accept`:** Format(s) the client wants back (`Accept: application/json;q=0.9, application/xml;q=0.5`). If unsupported, server returns **`406 Not Acceptable`**; if request `Content-Type` is unsupported, server returns **`415 Unsupported Media Type`**.
- **`Authorization`:** `Bearer <JWT>` carrying caller/user identity.
- **`Idempotency-Key`:** Client-generated UUID for safely retrying `POST` mutations.
- **`traceparent` & `tracestate`:** W3C standard headers for distributed tracing across hops.
- **`X-Correlation-ID` / `X-Request-ID`:** Cross-service request identifier for log correlation.
- **`Cache-Control`, `ETag`, `If-None-Match`:** HTTP caching and optimistic concurrency (`If-Match` returning `412 Precondition Failed`).
- **`Retry-After`:** Tells client how many seconds to wait after `429` or `503`.

---

#### 7. JSON Serialization and Deserialization

In Java microservices (Spring Boot / Quarkus), **Jackson (`ObjectMapper`)** converts POJOs/records to JSON and back.

##### Critical Microservice Serialization Rules
1. **Ignore Unknown Properties (`@JsonIgnoreProperties(ignoreUnknown = true)` / `DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES = false`):** Essential for **forward compatibility**—when a producer adds a new JSON field, older consumers must not crash!
2. **Avoid `double`/`float` for Currency:** Always serialize monetary amounts as `BigDecimal` (or integer minor units like `cents` + ISO currency code) to avoid floating-point rounding bugs (`0.1 + 0.2 != 0.3`).
3. **Standardize Timestamps as ISO-8601 UTC (`Instant` / `OffsetDateTime`):** Register `JavaTimeModule`, disable `WRITE_DATES_AS_TIMESTAMPS`, and serialize dates as `"2026-11-19T13:05:00Z"` instead of epoch arrays or timezone-less strings.
4. **Never Serialize Internal JPA Entities Directly:** Always map to explicit DTOs/records so lazy-loading proxies (`LazyInitializationException`), bidirectional cycles, or sensitive fields (`passwordHash`) never leak.
5. **Mind Serialization Overhead:** Reflection-heavy Jackson parsing on large payloads burns significant CPU and allocates short-lived GC objects—benchmark with **Blackbird** / Quarkus compile-time Jackson or switch to **Protobuf (gRPC)** for internal high-QPS traffic.

---

#### 8. REST API Design Principles

- **Use Nouns, Not Verbs:** `/api/v1/orders/{orderId}/items` instead of `/getOrderById` or `/createNewOrder`.
- **Use Plural Collection Names:** `/customers`, `/orders`, `/payments` for consistency.
- **Keep Hierarchy Shallow:** Limit nesting to 1–2 levels (`/customers/{id}/orders` is good; `/orgs/{o}/depts/{d}/teams/{t}/users/{u}/tasks` is hard to maintain—promote sub-resources to top-level endpoints).
- **Use Sub-Resource Action Endpoints When Needed:** For non-CRUD state transitions, model an action sub-resource (`POST /orders/{id}/cancel`, `POST /payments/{id}/capture`) rather than abusing `PUT /orders/{id}` with side-effect flags.
- **Avoid Over-Fetching / Chatty Calls:** Support sparse fieldsets or sensible summary DTOs for list endpoints (`GET /orders` returns summaries; `GET /orders/{id}` returns full details).

---

#### 9. REST versus gRPC

| Dimension | REST (HTTP + JSON) | gRPC (HTTP/2 + Protobuf) |
| --- | --- | --- |
| **Protocol** | HTTP/1.1 or HTTP/2 | Requires **HTTP/2** (multiplexing, header compression) |
| **Payload Format** | Human-readable text (**JSON**) | Compact binary (**Protocol Buffers**) — 3x–10x smaller & faster to serialize |
| **Contract Definition** | Optional (OpenAPI / Swagger YAML) | **Strict Contract-First** (`.proto` IDL generates client & server stubs) |
| **Communication Modes** | Request–Response (plus SSE/WebSockets) | **Unary, Server Streaming, Client Streaming, Bidirectional Streaming** |
| **Performance & CPU** | Higher serialization CPU & bandwidth | Very low latency, minimal CPU/GC overhead |
| **Browser Support** | Native in all browsers | Requires gRPC-Web proxy (no native browser support) |
| **Debugging / Tooling** | Easy (`curl`, browser, human-readable logs) | Harder (`grpcurl`, Postman gRPC, schema needed to inspect wire bytes) |
| **Best Use Cases** | Public/external APIs, BFF-to-UI, simple CRUD services | **Internal high-throughput microservice-to-microservice RPC**, polyglot services, streaming |

---

#### 10. REST versus GraphQL

| Dimension | REST | GraphQL |
| --- | --- | --- |
| **Endpoints** | Multiple resource URIs (`GET /users/1`, `GET /orders`) | Single endpoint (`POST /graphql`) |
| **Over-fetching / Under-fetching** | Fixed server payloads (may return unused fields or require multiple calls) | Client requests **exact fields needed** in a single query |
| **Caching** | Built-in HTTP caching (`GET`, CDN, `ETag`, `Cache-Control`) | Complex (everything is `POST` by default; requires Apollo/client normalized cache or persisted GET queries) |
| **Backend Pitfalls** | Chatty clients making $N$ sequential requests | **N+1 query problem** in nested resolvers (must mitigate with **`DataLoader` batching**) & malicious deep-query DoS (needs query depth/cost limiting) |
| **File Uploads & Errors** | Native HTTP multipart + standard status codes | Always returns `200 OK` with `"errors": [...]` array; file uploads need extra spec |
| **Best Use Cases** | Domain microservices, public APIs, cacheable reads | **BFF / API Aggregation layer** serving diverse Mobile & Web UIs |

---

#### 11. OpenFeign

**OpenFeign** (Spring Cloud OpenFeign / Quarkus MicroProfile Rest Client `@RegisterRestClient`) is a **declarative HTTP client** where you define a Java interface with annotations, and the framework generates the HTTP implementation at runtime/build time.

```java
@FeignClient(name = "inventory-service", url = "${clients.inventory.url}")
public interface InventoryClient {
    @GetMapping("/api/v1/inventory/{sku}")
    InventoryResponse getStock(@PathVariable("sku") String sku);
}
```

- **Pros:** Zero boilerplate, clean interface abstraction, easy integration with Spring Cloud LoadBalancer, Resilience4j circuit breakers, and interceptors (propagating JWT / trace headers).
- **Important Pitfalls:**
  - By default, classic Feign used `HttpURLConnection` (no connection pooling!). Always configure **Apache HttpClient 5** or **OkHttp** underneath.
  - **Synchronous & blocking:** Hides network I/O behind what looks like a local Java method call—easy for developers to accidentally call inside loops!
  - Note: Spring Cloud OpenFeign is feature-complete/maintenance-focused in favor of **Spring 6 HTTP Interface (`@HttpExchange`) + `RestClient`/`WebClient`**.

---

#### 12. RestClient

Introduced in **Spring Framework 6.1 / Spring Boot 3.2**, **`RestClient`** is the modern fluent synchronous replacement for legacy `RestTemplate`.

```java
RestClient restClient = RestClient.builder()
    .baseUrl("http://payment-service:8080")
    .requestFactory(httpComponentsClientHttpRequestFactory()) // Apache HttpClient 5 pool + timeouts
    .build();

PaymentResponse response = restClient.post()
    .uri("/api/v1/payments")
    .header("Idempotency-Key", idempotencyKey)
    .body(paymentRequest)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
        throw new PaymentRejectedException(res.getStatusCode());
    })
    .body(PaymentResponse.class);
```

- **Why it shines with Java 21+ Virtual Threads:** Enabling `spring.threads.virtual.enabled=true` lets `RestClient` write clean imperative blocking code while scaling to thousands of concurrent outbound calls without exhausting OS platform threads (unlike Tomcat's old ~200 thread limit).

---

#### 13. WebClient

**`WebClient`** (from `spring-boot-starter-webflux`, built on Project Reactor & Netty) is a **non-blocking, reactive HTTP client** returning `Mono<T>` (0..1 item) or `Flux<T>` (0..N stream).

- When `WebClient` sends a network request, the Netty event-loop thread does **not** block waiting for the response; it handles other requests and invokes a callback when response bytes arrive.
- Great for **parallel fan-out aggregation** (`Mono.zip(orderMono, userMono, paymentMono)`) and streaming backpressure.

##### Java HTTP Clients Comparison
| Client | Style | Thread Model | Status / Recommendation |
| --- | --- | --- | --- |
| **`RestTemplate`** | Imperative template | Blocking | Legacy (maintenance mode); avoid for new services |
| **`OpenFeign`** | Declarative interface | Blocking | Widely used; ensure Apache HttpClient 5 pooling is enabled |
| **`RestClient`** (`+ @HttpExchange`) | Modern fluent / declarative | Blocking *(scales with Java 21 Virtual Threads)* | **Default choice** for Spring Boot 3.2+ non-reactive services |
| **`WebClient`** | Fluent reactive (`Mono`/`Flux`) | Non-blocking event loop (Netty) | Best for Spring WebFlux stacks, streaming, or high-concurrency async fan-out |

---

#### 14. Synchronous Communication Limitations

Relying heavily on synchronous HTTP/gRPC calls across multiple services introduces severe architectural risks:

1. **Temporal Coupling:** Both caller and callee (and every service in the chain) must be **simultaneously alive and responsive** at request time.
2. **Compounding Availability Loss:** If a request synchronously depends on 5 services, each at 99.9% uptime:
   $$\text{Combined Availability} = 0.999^5 \approx 99.5\% \text{ (5x more downtime!)}$$
3. **Latency Accumulation & Tail Latency Amplification:** Total latency is the sum of sequential hops (or `max()` of parallel fan-out hops), rapidly inflating **P99 latency**.
4. **Thread & Connection Pool Exhaustion:** If one downstream service slows down from 20ms to 5s, caller threads pile up waiting—starving the caller's Tomcat thread pool and HTTP connection pool, triggering **cascading failures**.
5. **Tight Runtime Coupling:** Adding a new downstream consumer (e.g., `AuditService` or `LoyaltyService`) forces modifying and redeploying `OrderService` to call it synchronously.

---

### 2.2 HTTP and Networking Internals

---

#### 1. DNS Resolution

Before an HTTP client can open a TCP socket to `http://payment-service:8080`, it must translate the hostname into an IP address:
1. Client checks JVM/OS DNS cache (`/etc/hosts`, local resolver).
2. If missed, sends UDP/TCP query to the cluster DNS server (**CoreDNS** in Kubernetes).
3. **Kubernetes `ndots:5` Pitfall:** By default in K8s `/etc/resolv.conf`, `options ndots:5` causes a lookup for `api.stripe.com` (only 2 dots) to first search `api.stripe.com.namespace.svc.cluster.local`, `...svc.cluster.local`, `...cluster.local`, generating 4–5x extra DNS queries (and potential `conntrack` UDP drops / 5-second CoreDNS timeouts) unless you use a trailing dot (`api.stripe.com.`) or lower `ndots`.

---

#### 2. TCP Connection Establishment

HTTP/1.1, HTTP/2, and gRPC run over **TCP** (Transmission Control Protocol), which provides ordered, reliable, byte-stream delivery.
- Opening a fresh TCP connection requires the OS kernel to allocate a socket descriptor, assign an **ephemeral source port**, perform the 3-way handshake (and TLS handshake), and run TCP slow-start congestion control.
- Creating a brand-new connection per HTTP request adds massive latency and exhausts OS sockets under load.

---

#### 3. TCP Three-Way Handshake

Establishes synchronized sequence numbers between client and server—taking **1 full network Round-Trip Time (RTT)** before any application or TLS data is sent:

```text
Client                                   Server
  │ ─────── SYN (seq = x) ─────────────────► │  1. Client initiates connection
  │ ◄────── SYN-ACK (seq = y, ack = x+1) ─── │  2. Server acknowledges & sends its seq
  │ ─────── ACK (ack = y+1) ───────────────► │  3. Connection ESTABLISHED (1 RTT done)
  │ ═══════ TLS / HTTP Data Flows ══════════ │
```

- **SYN Backlog Exhaustion:** Under massive burst traffic, the server's OS accept queue (`somaxconn` / `tcp_max_syn_backlog`) can fill up, silently dropping new `SYN` packets and causing client connection timeouts.

---

#### 4. TLS Handshake

For HTTPS / mTLS, after TCP establishes, the **TLS Handshake** authenticates identities (X.509 certificates) and negotiates symmetric encryption keys:
- **TLS 1.2:** Adds **2 extra RTTs** on top of the TCP handshake (total **3 RTTs** before the first HTTP byte!).
- **TLS 1.3:** Streamlined to **1 extra RTT** (and **0-RTT** with session resumption)—dramatically reduces cold-connection latency.
- **mTLS (Mutual TLS):** Both server **and** client present and verify certificates, establishing strong cryptographic service-to-service identity.

---

#### 5. HTTP/1.1 versus HTTP/2

| Feature | HTTP/1.1 | HTTP/2 |
| --- | --- | --- |
| **Framing** | Plain text | **Binary frames** split into multiplexed **streams** |
| **Concurrency per TCP Connection** | **1 active request at a time** (HTTP Head-of-Line blocking; clients must open 6+ parallel TCP connections) | **Many concurrent requests/responses multiplexed** over **1 single TCP connection** |
| **Header Compression** | None (repeats large `Authorization` / `Cookie` headers every call) | **HPACK** stateful header compression |
| **Streaming** | Chunked transfer encoding (one direction well) | Native full-duplex bidirectional streams (required by **gRPC**) |
| **TCP Head-of-Line Blocking** | Affected (packet loss stalls that connection) | Still affected at **TCP layer** (one lost TCP packet stalls all multiplexed HTTP/2 streams on that connection—solved by HTTP/3 QUIC) |
| **K8s L4 Load Balancing Trap** | Opening new connections spreads across pods | **Connection pinning!** Because HTTP/2 reuses 1 long-lived TCP connection, a Kubernetes L4 `ClusterIP` sends *all* requests to **one pod** unless you use an L7 proxy / Service Mesh (Envoy/Istio/Linkerd) or gRPC client-side load balancing! |

---

#### 6. HTTP Keep-Alive

**HTTP Keep-Alive** (`Connection: keep-alive` — default in HTTP/1.1 and implicit in HTTP/2) instructs client and server **not to close the underlying TCP/TLS socket** after a single HTTP response completes.
- Server response may advertise `Keep-Alive: timeout=60, max=1000`, meaning it will hold the idle socket for up to 60 seconds or 1000 requests before closing it.

---

#### 7. Connection Reuse

Instead of executing `DNS → TCP 3-way handshake → TLS handshake` on every call, **connection reuse** allows the client to check out an already-open, authenticated, warm TCP/TLS socket from memory, write the next HTTP request immediately (`0` handshake RTTs), and benefit from an already-expanded TCP congestion window.

---

#### 8. Connection Pooling

An **HTTP Connection Pool** (e.g., Apache HttpClient 5 `PoolingHttpClientConnectionManager`, Netty `ConnectionProvider`, OkHttp `ConnectionPool`) maintains bounded sets of reusable connections keyed by target route `(scheme, host, port)`.

##### Critical Tuning Parameters (Production Trap!)
1. **`maxTotal`:** Maximum concurrent connections across all downstream destinations combined.
2. **`defaultMaxPerRoute` (Max Connections Per Host):**
   - **Classic Java Gotcha:** Apache HttpClient's default `maxPerRoute` was historically **2** (or 5 in newer setups) and `maxTotal` was **20**! If your microservice calls `PaymentService` with 50 concurrent threads, 48 threads block waiting on just 2 connections! Always tune `maxPerRoute` (e.g., 50–200 based on load tests).
3. **Idle Connection Eviction & Max Lifetime:** Periodically close idle connections **before** the server or cloud NAT/ALB closes them, and rotate connections periodically (`maxLifeTime`) so traffic shifts cleanly across new autoscaled pods/DNS entries.
4. **Stale Connection Validation:** Briefly check if a pooled socket is still open before reusing it after idle periods.

---

#### 9. Persistent Connections

A **persistent connection** stays open across multiple sequential (HTTP/1.1) or concurrent (HTTP/2) requests.

##### The Stale Half-Closed Connection Race Condition
- Suppose the **server** (or an intermediate load balancer like AWS ALB / NGINX) has an idle timeout of **60s**, and the **client** pool keeps idle connections for **65s**.
- At `t = 60.001s`, the server sends TCP `FIN` to close the idle socket while the client simultaneously grabs it from the pool and sends `POST /pay`.
- The server responds with TCP `RST`, causing `NoHttpResponseException` or `Connection reset`.
- **Golden Rule:** Always configure **client pool idle timeout strictly shorter** than the server/LB idle timeout (e.g., server = 60s, client idle eviction = 30–45s).

---

#### 10. Socket Exhaustion

**Socket exhaustion** happens when a service opens and closes TCP connections faster than the OS can reclaim them.

- A TCP socket is identified by the 4-tuple `(srcIP, srcPort, dstIP, dstPort)`.
- Linux allocates ephemeral `srcPort` values from a finite range (`net.ipv4.ip_local_port_range`, default ~`32768–60999`, i.e., **~28,232 ports**).
- When a client closes a TCP connection actively, the OS keeps that socket in **`TIME_WAIT`** state for **2 × MSL (~60 seconds)** to handle delayed network packets.
- **Root Cause:** Creating a `new RestTemplate()` / `new HttpClient()` per request instead of reusing a singleton pooled client! At ~500 RPS without pooling, all 28k ephemeral ports enter `TIME_WAIT` within 60 seconds → `java.net.BindException: Cannot assign requested address`.

---

#### 11. Connection Pool Exhaustion

**Connection pool exhaustion** occurs inside the application's HTTP client pool (or database HikariCP pool) when **all `maxPerRoute` / `maxTotal` slots are actively leased** and waiting requests exceed the `connectionRequestTimeout`.

##### Root Causes
1. **Downstream service hangs** with missing or overly generous Read Timeout, holding connections indefinitely.
2. **Pool limits too small** (`defaultMaxPerRoute` left at default 2–5).
3. **Connection Leak:** Opening a streaming HTTP response (`InputStream`) and forgetting to read/close the `Response` body so the connection is never returned to the pool.

- **Symptom:** Sudden spike in `ConnectionPoolTimeoutException` / `AcquireTimeoutException` even while downstream CPU is low or normal.

---

#### 12. Connection Timeout

**Connection Timeout** (`connectTimeout`) is the maximum time to wait for the **TCP 3-way handshake** (and optionally TLS handshake) to complete with the destination IP.
- **When it triggers:** Downstream pod is down, firewall silently drops `SYN` packets, or host IP is unreachable.
- **Recommended value:** **Fail fast** (`500ms – 2000ms`), because a healthy internal service responds to `SYN` in `< 5ms`.

---

#### 13. Read Timeout

**Read Timeout** (`socketTimeout` / `responseTimeout`) is the maximum gap allowed **between receiving consecutive data packets** once the TCP connection is established.
- **When it triggers:** Downstream accepted the connection, then hung on a slow SQL query, deadlock, or Stop-the-World GC pause without sending bytes.
- **Pitfall:** Read timeout only caps **inactivity between packets**, not total response duration! A trickle-feeding server sending 1 byte every second defeats a 2s read timeout.

---

#### 14. Write Timeout

**Write Timeout** is the maximum time allowed to push outbound request bytes into the OS TCP send buffer when sending data to the server.
- **When it triggers:** Uploading large payloads (files, big batch JSON) while the receiver stops reading (TCP zero-window / backpressure) or network congestion stalls ACKs.

---

#### 15. Request Timeout

**Request Timeout** (Total / End-to-End Call Deadline) caps the **entire wall-clock duration** of an outbound call: pool wait + DNS + TCP connect + TLS + request write + server processing + reading the full response body.

```text
|<────────────────────────── Request Timeout (e.g., 2000ms) ──────────────────────────>|
| Pool Lease | DNS + Connect | TLS | Write Body | Server Thinks... | Read Full Body    |
|<── 100ms ─>|<─── 500ms ───>|     |<── 500ms ─>|                  |<─ Read Timeout ─>|
```

- **Best Practice:** Always configure **all three**: Pool acquisition timeout, Connection timeout, AND an overall Request/Call deadline (e.g., Resilience4j `TimeLimiter` or Netty `responseTimeout`).

---

#### 16. DNS Caching

DNS lookups can be cached by the OS (`nscd` / `systemd-resolved`), JVM, or HTTP client:
- **JVM Pitfall:** Historically, Java's `InetAddress` cached successful lookups forever (`-1`) when a SecurityManager was present, or defaults to `networkaddress.cache.ttl=30` seconds (check your JDK version and `java.security`).
- **Two Extremes:**
  - **Caching too long (`-1` / infinite):** When Kubernetes replaces pods or AWS ALB rotates IPs, your service keeps calling dead IP addresses!
  - **Caching disabled (`0`):** Every HTTP call hammers CoreDNS, adding latency and risking CoreDNS saturation.
- **Best Practice:** Use a short TTL (e.g., **5–30 seconds**) and `networkaddress.cache.negative.ttl=0` or `5` so failed lookups recover quickly.

---

#### 17. Connection Reset

**Connection Reset** (`java.net.SocketException: Connection reset` / `ECONNRESET`) happens when the remote endpoint abruptly terminates an established TCP connection by sending a **`RST`** packet instead of a graceful `FIN`.

##### Common Microservice Causes
1. Downstream pod **crashed or was OOMKilled** mid-request (no graceful shutdown).
2. Client reused an **idle pooled connection** that the server/load balancer had already silently closed.
3. Intermediate firewall / Kubernetes `kube-proxy` conntrack entry expired on a long-idle socket.

---

#### 18. Broken Pipe

**Broken Pipe** (`java.io.IOException: Broken pipe` / `EPIPE`) happens when a process **writes bytes into a socket whose peer has already closed its end** (receiving a `RST` after writing to a closed socket).

##### Classic Microservice Scenario
1. Upstream `Gateway` calls `ReportService` with a **2-second timeout**.
2. At `t = 2.0s`, `Gateway` times out and closes the TCP connection.
3. `ReportService` wasn't cancelled, finishes its heavy SQL query at `t = 5.0s`, and tries to flush its HTTP JSON response to the socket → **Boom: `Broken pipe` in `ReportService` logs!**

---

#### 19. Network Partitions

A **network partition** occurs when a network failure (switch failure, misconfigured NetworkPolicy, AZ isolation, asymmetric packet loss) splits services so they cannot reach each other even though both are running fine.
- **Why it is tricky:** Service A cannot tell whether Service B **crashed before executing**, **crashed after committing**, or **committed and its response packet was lost in the partition**.
- Drives the necessity for **CAP trade-offs**, **idempotency keys**, and **circuit breakers**.

---

#### 20. Network Latency

**Network latency** is the time taken for a message to travel between caller and callee, composed of:
- Physical propagation delay (cross-AZ ~1–2ms, cross-region ~50–150ms)
- Serialization/deserialization CPU time + TLS encryption
- OS kernel queuing + proxy hops (API Gateway + Ingress + Envoy sidecar proxies!)

**Why P99 Matters More Than Mean Latency:** If a webpage renders by calling 10 downstream microservices in parallel, and each service has a 1% chance of a slow response (P99 = 500ms), the chance that the *user* experiences ≥500ms latency on that page load is:
$$1 - 0.99^{10} \approx 9.6\% \text{ (almost 1 in 10 requests!)}$$

---

#### 21. Request Propagation Across Services

When a single user action crosses 5+ services and Kafka topics, critical metadata must flow seamlessly across every hop:
- **Distributed Tracing Context:** W3C `traceparent` and `tracestate` headers (propagated automatically by OpenTelemetry / Micrometer Tracing—and explicitly injected into Kafka message headers!).
- **Correlation / Request ID:** `X-Correlation-ID` / `X-Request-ID` mapped into SLF4J **MDC** (`MDC.put("traceId", ...)`).
- **Security Context:** Propagating end-user JWT (`Authorization: Bearer ...`) or exchanged internal service token.
- **Timeout / Deadline Budget:** Passing remaining time budget (e.g., `grpc-timeout` or `X-Request-Deadline`) so downstream services abort work if the upstream caller has already expired!
- **Thread-Local Gotcha:** In Java async code (`CompletableFuture`, `@Async`, Reactive `WebClient`), `ThreadLocal` MDC/SecurityContext is **lost across thread switches** unless you wrap executors (`ContextSnapshot` / Micrometer Context Propagation / Reactor Hooks).

---

### 2.3 Asynchronous Communication

---

#### 1. Message Brokers

A **message broker** is middleware sitting between producers and consumers that accepts, stores, routes, and delivers messages asynchronously—decoupling services in **time** (consumer doesn't have to be online when producer publishes) and **space** (producer doesn't know consumer IPs or count).

| Feature | Apache Kafka | RabbitMQ (AMQP) |
| --- | --- | --- |
| **Core Model** | Distributed **append-only immutable commit log** partitioned by key | **Smart broker / dumb consumer** (Exchanges route messages into per-consumer Queues) |
| **Consumption Style** | **Pull-based** (consumers pull batches & track offsets) | **Push-based** (broker tracks per-message acks & deletes once acked) |
| **Retention & Replay** | Messages **retained by time/size** (e.g., 7 days)—**multiple consumer groups can replay** history anytime | Messages **deleted immediately** once acknowledged (not meant for historical replay) |
| **Ordering** | Strictly ordered **within each partition** (use `orderId` as partition key) | FIFO within a single queue with 1 consumer (breaks with multiple competing consumers + retries) |
| **Throughput** | Extremely high (hundreds of thousands to millions msg/sec via zero-copy I/O) | Moderate–high (tens of thousands msg/sec; flexible complex routing rules) |
| **Best For** | Event streaming, EDA, event sourcing, CDC, high-throughput pipelines, audit replay | Task/work queues, complex per-message routing, priority queues, delayed jobs, RPC over queues |

---

#### 2. Message Queues

A **message queue** implements **point-to-point (competing consumers)** communication:
- Producer sends a message to a queue.
- Multiple worker instances listen to the queue, and **each message is delivered to and processed by only ONE consumer instance** (horizontal scaling for background work).
- In Kafka, this is achieved by putting multiple replicas of the same microservice into **the same `consumerGroupId`** (1 partition assigned to at most 1 consumer within the group).

```text
                    ┌──► Worker 1 (processes Msg 1, 3)
Producer ──► [Queue]┼──► Worker 2 (processes Msg 2, 5)
                    └──► Worker 3 (processes Msg 4)
```

---

#### 3. Publish-Subscribe Architecture

**Publish–Subscribe (Pub/Sub)** implements **1-to-many fan-out**:
- Producer publishes an event *once* to a topic/exchange without knowing who is listening.
- Multiple independent services subscribe and **each receives its own copy** of the event.
- In Kafka: Give each service its own distinct **`consumerGroupId`** (`inventory-group`, `notification-group`, `analytics-group`) subscribing to the same `order-events` topic.

```text
                                 ┌──► [Inventory Group]     ──► Reserve Stock
OrderService ──► (order-events) ─┼──► [Notification Group]  ──► Send Confirmation Email
                                 └──► [Analytics Group]     ──► Update Sales Dashboard
```

---

#### 4. Event-Driven Architecture (EDA)

**Event-Driven Architecture** is a paradigm in which microservices communicate primarily by producing and reacting to **events** (state changes) rather than direct point-to-point RPC calls.

- **Benefits:** Extreme decoupling (add a new `FraudAnalyticsService` consumer without touching `OrderService`), load leveling (Kafka absorbs traffic spikes so consumers process at steady DB pace), and high runtime autonomy.
- **Trade-offs:** Harder to trace end-to-end execution flow, eventual consistency, duplicate handling, and monitoring consumer lag.

---

#### 5. Commands versus Events

| Dimension | Command | Event |
| --- | --- | --- |
| **Intent** | Request to **perform an action** in the future | Notification of a **fact that already happened** |
| **Naming** | Imperative verb (`ReserveInventory`, `ChargeCard`) | Past tense (`InventoryReserved`, `PaymentCharged`) |
| **Coupling** | **Directed** at a specific target service (point-to-point) | **Broadcast** to zero or more unknown subscribers (pub/sub) |
| **Can It Be Rejected?** | **Yes** (target can fail validation / return error) | **No** (you cannot reject history; consumer decides whether/how to react) |
| **Delivery Channel** | REST/gRPC or dedicated Command Queue | Pub/Sub Topic (`order-events`) |

---

#### 6. Event Notification

In **Event Notification**, a producer publishes a **thin, minimal event** containing just the event type and entity ID:
```json
{ "eventId": "e-91", "eventType": "OrderPlaced", "orderId": "ord-778" }
```
- If `BillingService` needs line items or customer address, it **calls `GET /orders/ord-778` back on `OrderService`**.
- **Pros:** Tiny broker payloads; consumer always fetches the latest state; avoids leaking sensitive fields onto shared Kafka topics.
- **Cons:** Creates **callback storms** when thousands of events trigger synchronous HTTP calls back to the producer, reducing runtime autonomy.

---

#### 7. Event-Carried State Transfer (ECST)

In **Event-Carried State Transfer**, the producer includes **all relevant changed state** directly in the event payload:
```json
{
  "eventId": "e-91",
  "eventType": "CustomerAddressUpdated",
  "customerId": "cust-101",
  "shippingAddress": { "city": "Austin", "zip": "78701" }
}
```
- `OrderService` and `ShippingService` store a **local read copy** of the customer's address in their own DB and **never call `CustomerService` at runtime**.
- **Pros:** Maximum runtime autonomy, zero synchronous network latency on reads, immune to `CustomerService` downtime.
- **Cons:** Larger message payloads, duplicated data across DBs, eventual consistency lag, and schema coupling to payload fields.

---

#### 8. Event Choreography

**Event Choreography** coordinates multi-service business workflows without a central controller—each service listens to upstream events, executes its local transaction, and publishes its own domain event for the next step:
```text
OrderService ──[OrderPlaced]──► PaymentService ──[PaymentSucceeded]──► InventoryService ──[StockReserved]──► ShippingService
```
- **Pros:** No single point of failure or central coordinator bottleneck; services stay decoupled.
- **Cons:** Workflow logic is scattered across many services—hard to understand the whole flow, debug stuck sagas, or add conditional branching across 5+ steps (where **Saga Orchestration** is cleaner).

---

#### 9. Asynchronous Processing

Used when an HTTP client triggers slow, bursty, or non-critical work (generating PDFs, sending emails, processing video, bulk imports):
1. Client sends `POST /reports`.
2. API validates input, enqueues job to Kafka/RabbitMQ (or Transactional Outbox), and immediately returns **`202 Accepted`** with `Location: /reports/job-42`.
3. Background workers process the job asynchronously.
4. Client learns completion via **polling** (`GET /reports/job-42`), **Webhooks**, **Server-Sent Events (SSE)**, or **WebSockets**.

---

#### 10. Message Acknowledgements

An **Acknowledgement (ACK)** tells the broker that the consumer has safely processed a message and its offset/delivery can advance.

- **Auto-Ack / Auto-Commit (Dangerous!):** Broker marks message delivered as soon as it hits the consumer's memory buffer. If the consumer pod crashes or hits a DB error 10ms later, **the message is permanently lost**.
- **Manual Ack (`AckMode.MANUAL_IMMEDIATE` in Spring Kafka):** Consumer executes business logic + commits its DB transaction **first**, and only then commits the offset / sends `ACK` to the broker.
- **RabbitMQ `NACK` / `REJECT`:** `requeue=true` puts the message back on the queue (watch out for infinite tight retry loops!); `requeue=false` routes it directly to a **Dead-Letter Exchange (DLX)**.

---

#### 11. Message Delivery Guarantees

| Guarantee | Behavior | How It Happens | Practical Reality in Microservices |
| --- | --- | --- | --- |
| **At-most-once** | Message delivered **0 or 1** times *(never duplicates, can lose data)* | Producer `acks=0`, or consumer auto-acks before processing | Only acceptable for non-critical metrics/telemetry/logs |
| **At-least-once** | Message delivered **1 or more** times *(never lost, may duplicate)* | Producer retries (`acks=all`), consumer manually acks **after** DB commit | **Industry default!** If consumer crashes *after* DB commit but *before* ACK, broker redelivers → **consumers MUST be idempotent** |
| **Exactly-once** | Every message processed **exactly 1 time** | True distributed exactly-once across external DB + broker is impossible without 2PC; Kafka supports EOS (`enable.idempotence=true` + transactions) **only inside Kafka-to-Kafka pipelines** | Achieved in practice as **Effectively-Once Processing = At-least-once delivery + Idempotent Consumer (Transactional Inbox)** |

---

#### 12. Dead-Letter Queues (DLQ)

A **Dead-Letter Queue / Topic** (`orders.events.DLQ`) stores messages that cannot be processed even after configured retries—preventing a single malformed or buggy message (**"poison pill"**) from blocking an entire Kafka partition or starving RabbitMQ workers.

##### Poison Pill Causes & Handling Playbook
- **Causes:** Unparseable JSON/Avro schema, null pointer bug on an edge-case payload, permanent downstream constraint violation.
- **Best Practice:**
  1. Retry transient failures locally with backoff (or non-blocking retry topics like `orders-retry-5s`, `orders-retry-1m`).
  2. After max retries (or immediately on non-retryable deserialization errors), forward to `.DLQ` **with diagnostic headers** (`original-topic`, `exception-message`, `stacktrace`, `timestamp`).
  3. **Alert** on DLQ depth > 0, inspect/fix the bug or payload, and **replay** DLQ messages safely via idempotent consumers.

---

#### 13. Message Replay

Because Kafka retains messages on disk for days/weeks (or compacted topics retain latest keys indefinitely), **message replay** lets you rewind consumer group offsets back to an earlier timestamp or offset `0` to:
- Rebuild a corrupted **CQRS materialized view**.
- Hydrate a **brand-new microservice's database** with historical state.
- Reprocess events after fixing a consumer bug.

> **Replay Pitfall (Side-Effect Explosion):** Make sure replaying old events doesn't send 50,000 duplicate customer emails or re-charge credit cards! Protect external side effects with **idempotency checks** or **replay-mode flags**.

---

#### 14. Event Schema Design

Well-designed events separate standard routing **Envelope Metadata** from business **Payload Data**:

```json
{
  "eventId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "eventType": "OrderPlaced",
  "eventVersion": "1.0",
  "occurredAt": "2026-11-19T13:05:00Z",
  "producer": "order-service",
  "correlationId": "req-88912",
  "aggregateId": "ord-778",
  "data": {
    "orderId": "ord-778",
    "customerId": "cust-101",
    "totalAmount": "149.99",
    "currency": "USD"
  }
}
```
- **Schema Formats:** JSON Schema (human-readable, flexible), **Apache Avro** (compact binary, required by many Kafka Schema Registries, great evolution rules), or **Protobuf** (strict typed, fast).
- **Schema Registry (Confluent / Apicurio):** Central repository validating schemas before producers publish and distributing schemas by ID to consumers.

---

#### 15. Event Versioning

When an event's structure must change in a **breaking** way (e.g., changing `amount` from `"100.00"` string to `{ "value": 10000, "currency": "USD" }` or splitting `fullName` into `firstName`/`lastName`):
1. **Prefer Additive Non-Breaking Evolution:** Add a new optional field (`loyaltyDiscount`) while keeping existing fields intact—no version bump needed.
2. **Versioned Event Types / Schemas:** Introduce `OrderPlacedV2` (or `eventVersion: "2.0"`).
3. **Dual-Publishing (Transition Window):** Producer temporarily emits both V1 and V2 events (or publishes V2 while V1 topic drains) until all consumers migrate, then retires V1.
4. **Upcasting (Consumer-Side):** For Event Sourcing stores where old events are immutable, write an **Upcaster** that transforms stored `V1` events into `V2` objects in memory during replay.

---

#### 16. Schema Evolution

**Schema evolution** allows producers and consumers—deployed at completely different times by independent teams—to upgrade event schemas without breaking deserialization or causing downtime.

- Enforced automatically in CI/CD and at runtime by a **Schema Registry** using explicit compatibility rules (`BACKWARD`, `FORWARD`, `FULL`, `_TRANSITIVE`).

---

#### 17. Backward and Forward Compatibility

| Compatibility Mode | Meaning | Allowed Schema Changes | Safe Deployment Order |
| --- | --- | --- | --- |
| **Backward Compatible** | **New** consumer schema can read **old** producer messages | Add **optional field with default**; delete a field | **Upgrade Consumers FIRST**, then Producers |
| **Forward Compatible** | **Old** consumer schema can read **new** producer messages | Add field (old consumer ignores unknown fields via `@JsonIgnoreProperties(ignoreUnknown = true)`); delete optional field | **Upgrade Producers FIRST**, then Consumers |
| **Full Compatible** | Both **Backward AND Forward** compatible | **Only** add or remove **optional fields with default values** (never rename, delete required fields, or change field types!) | **Deploy Producers and Consumers in ANY order** (gold standard for microservices!) |

---

### 2.4 API Design

---

#### 1. API Contracts

An **API Contract** is the formal, binding agreement between a service provider and its consumers defining:
- Endpoints, HTTP methods, headers, and query parameters
- Request & response JSON/Protobuf schemas, types, nullability, and validation constraints
- Status codes and error response formats
- Non-functional SLAs (latency expectations, rate limits, auth scopes, idempotency behavior)

##### Contract-First vs Code-First
- **Contract-First:** Write `openapi.yaml` or `.proto` *first*, review with consumers, then generate server stubs and client SDKs (`openapi-generator`, `protoc`). Prevents accidental breaking changes and enables parallel frontend/backend work.
- **Code-First:** Write Spring `@RestController` / Quarkus resource classes with annotations and generate OpenAPI specs automatically (`springdoc-openapi`). Faster for small internal services, but requires CI contract diffs so code refactoring doesn't silently break the API.

---

#### 2. API Versioning

| Strategy | Example | Pros | Cons |
| --- | --- | --- | --- |
| **URI Path Versioning** | `GET /api/v1/orders` → `/api/v2/orders` | Most explicit, easy to route in API Gateway, browser-friendly, easy caching | Pollutes resource URI; encourages coarse breaking jumps |
| **Custom Header Versioning** | `X-API-Version: 2` | Clean URIs | Harder to test in browsers/links; proxies/CDNs need `Vary: X-API-Version` |
| **Media Type (Accept Header)** | `Accept: application/vnd.shop.order.v2+json` | Purest REST/HTTP content negotiation; supports per-resource versioning | Complex client setup; harder to inspect casually |
| **Query Parameter** | `GET /api/orders?version=2` | Simple to test | Mixes versioning with filtering params; optional defaults can surprise clients |

- **Industry Consensus:** **URI Path Versioning (`/v1/...`)** for major breaking external APIs (e.g., Stripe, GitHub, Twilio style) combined with **evolvable backward-compatible additive changes** so you rarely need `/v2`.

---

#### 3. Backward Compatibility

Following **Postel's Law (Robustness Principle)**: *"Be conservative in what you do (send), be liberal in what you accept from others."*

##### Safe (Non-Breaking) vs Breaking Changes
| Safe (Additive) Changes | Breaking Changes (Require New Version!) |
| --- | --- |
| Adding **new optional fields** to responses | Removing or renaming an existing JSON field |
| Adding **new optional** request fields with safe defaults | Adding a **new required** request field |
| Adding a **new endpoint** or new optional query param | Changing a field's **data type** (`"123"` → `123`) or date format |
| Relaxing validation (accepting larger string max-length) | Tightening validation (making optional field required, shrinking max length) |
| Adding new enum values *(only if clients handle unknown enums gracefully!)* | Changing success/error HTTP status semantics |

---

#### 4. Request Validation

Validate inputs aggressively at the **API boundary** before reaching domain logic or databases:
- **Syntactic Validation (`jakarta.validation`):** `@NotNull`, `@NotBlank`, `@Size`, `@Min`/`@Max`, `@Positive`, `@Email`, `@Pattern`, `@Valid` nested objects.
- **Semantic / Business Validation:** Executed inside Domain/Service layer (e.g., *"is account balance sufficient?"*, *"is discount code active?"*).
- Return **`400 Bad Request`** (with per-field errors) for malformed/missing inputs, and **`422 Unprocessable Entity`** or **`409 Conflict`** for domain rule violations.

---

#### 5. Pagination

Never return unbounded collections (`SELECT * FROM orders` returning 500,000 rows crashes JVMs with OOM!). Always enforce pagination with a sensible `maxLimit` (e.g., max 100).

| Aspect | Offset / Limit (`?page=5&size=20`) | Cursor / Keyset (`?after=ord_900&limit=20`) |
| --- | --- | --- |
| **SQL Implementation** | `ORDER BY created_at DESC LIMIT 20 OFFSET 100` | `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT 20` |
| **Performance at Deep Pages** | **Degrades ($O(N)$):** DB must scan and discard `100,000` rows for `OFFSET 100000` | **Constant ($O(1)$ index seek):** Fast even at page 1,000,000! |
| **Concurrent Inserts/Deletes** | **Drift bug:** New row inserted at top shifts items down, causing **duplicate items** across pages | **Stable:** Cursor anchors to exact item; no duplicates or skipped rows |
| **Random Page Jump (`Go to Page 7`)** | Supported | Not supported (`Next` / `Previous` only) |
| **Best For** | Small admin tables needing page numbers (`1, 2, 3...`) | Infinite feeds, mobile APIs, large transactional tables (`Orders`, `Ledger`, `Logs`) |

---

#### 6. Filtering and Sorting

- **Filtering:** Use explicit query parameters (`GET /orders?status=PAID&customerId=c-10&createdAfter=2026-01-01T00:00:00Z`).
- **Sorting:** Use consistent syntax like `?sort=-createdAt,totalAmount` (`-` prefix = descending).
- **Security & Performance Rule:** Always **allowlist** permitted filter and sort fields! Never pass raw query parameter column names directly into dynamic SQL `ORDER BY` (prevents **SQL Injection** and **full-table scans on unindexed columns**).

---

#### 7. Error Response Design

Poor error responses (`"Something went wrong"`, empty 500s, or leaked Java stack traces) slow down debugging and expose security internals.

##### Design Principles
- Never expose internal class names, SQL queries, or stack traces to callers.
- Provide a **stable machine-readable `errorCode`** (`INSUFFICIENT_INVENTORY`) so clients don't have to parse English error messages.
- Include **`traceId` / `correlationId`** in every error response so support/clients can report an ID you can immediately search in centralized logs and OpenTelemetry traces.

---

#### 8. Standardized Error Handling

Adopt **RFC 9457 (formerly RFC 7807) — Problem Details for HTTP APIs** (`Content-Type: application/problem+json`), built directly into Spring Boot 3 (`ProblemDetail`) and Quarkus.

```json
{
  "type": "https://api.example.com/errors/insufficient-stock",
  "title": "Insufficient Inventory",
  "status": 422,
  "detail": "Requested 5 units of SKU-101, but only 2 units are available.",
  "instance": "/api/v1/orders",
  "errorCode": "INVENTORY_INSUFFICIENT_STOCK",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "timestamp": "2026-11-19T13:05:10Z"
}
```

In Spring Boot 3, centralize all API exception mapping in a `@RestControllerAdvice` returning `ProblemDetail` for validation errors (`MethodArgumentNotValidException`), domain exceptions, and fallback 500s.

---

#### 9. API Documentation

- Documentation written manually in Confluence/wikis drifts out of date within weeks.
- Effective microservice API documentation must be **code/contract-driven and auto-published in CI**, covering:
  - Endpoints, schemas, required/optional fields, constraints, examples
  - Auth scopes/headers, rate limits, pagination format
  - Idempotency rules and all possible error codes (`400`, `401`, `403`, `409`, `422`, `429`)

---

#### 10. OpenAPI / Swagger

**OpenAPI (3.0 / 3.1)** is the industry-standard machine-readable YAML/JSON specification for HTTP REST APIs.
- **Java Integration:** `springdoc-openapi-starter-webmvc-ui` (Spring Boot) and `quarkus-smallrye-openapi` (Quarkus) generate `/v3/api-docs` and interactive Swagger UI automatically.
- **CI Governance:** Use tools like **Spectral** to lint OpenAPI specs in CI (enforcing naming conventions, security definitions, and RFC 9457 error responses) and **openapi-diff** to block accidental breaking changes on PRs.

---

#### 11. Consumer-Driven Contracts (CDC)

Traditional end-to-end integration tests spinning up 15 microservices are slow, flaky, and tightly coupled. **Consumer-Driven Contract Testing** (**Pact**, **Spring Cloud Contract**) verifies integration boundaries **independently**:

```text
1. Consumer Team (OrderService)
   Writes unit test mocking PaymentClient ─► Generates Pact Contract JSON ─► Uploads to Pact Broker

2. Provider Team (PaymentService)
   CI downloads Contracts from Pact Broker ─► Replays requests against real PaymentController
   ─► Fails PR immediately if PaymentService broke any Consumer expectation!
```

- **Key Benefit:** Catches breaking API or Kafka event schema changes in seconds inside CI without running live cross-service environments.

---

#### 12. API Deprecation Strategies

You cannot delete an API endpoint used by downstream microservices or mobile apps overnight without causing outages.

1. **Announce & Signal via Standard Headers (RFC 8594 / RFC 9745):**
   - `Deprecation: true` (or `@1763510400` date)
   - `Sunset: Wed, 01 Jul 2027 00:00:00 GMT`
   - `Link: <https://docs.example.com/migration-v2>; rel="deprecation"`
2. **Track Usage via API Gateway / Telemetry:** Monitor metrics per `client_id` / caller service to identify who is still calling `/api/v1/orders`.
3. **Proactive Outreach & Brownouts:** Notify remaining consumers; optionally run scheduled short **brownouts** (temporarily returning `410 Gone` for 15 minutes in staging/non-peak windows) to surface hidden undocumented callers.
4. **Retire Safely:** Once traffic reaches zero (or sunset date passes), return **`410 Gone`** (explicitly indicates permanent removal, unlike `404`), then decommission the route.

---

#### Quick Interview Revision Checklist (Section 2)
- Can you explain **Safe vs Idempotent HTTP methods** (`GET`, `PUT`, `DELETE`, `POST`, `PATCH`) and debug **`401` vs `403`** and **`502` vs `503` vs `504`**?
- Can you compare **REST vs gRPC vs GraphQL** and **RestTemplate vs OpenFeign vs RestClient vs WebClient** (especially with Java 21 Virtual Threads)?
- Can you trace a cold HTTP call (**DNS → TCP 3-way handshake → TLS → HTTP**) and explain **HTTP/2 connection pinning in Kubernetes**, **socket exhaustion (`TIME_WAIT`)**, **connection pool exhaustion**, **`Connection reset`**, and **`Broken pipe`**?
- Can you clearly differentiate **Connection Timeout**, **Read Timeout**, **Write Timeout**, and **Request Timeout**—and explain the **stale keep-alive connection race condition**?
- Can you contrast **Kafka vs RabbitMQ**, **Commands vs Events**, **Event Notification vs Event-Carried State Transfer (ECST)**, and **At-least-once + Idempotent Consumer**?
- Can you explain **Backward vs Forward vs Full schema compatibility** and which deployment order each requires?
- Can you compare **Offset vs Cursor pagination**, explain **RFC 9457 `ProblemDetail`**, and describe how **Consumer-Driven Contract Testing (Pact)** prevents breaking deployments?
