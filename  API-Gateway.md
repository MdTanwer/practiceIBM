# Microservices Study Notes — Section 3

## 3. API Gateway and Service Discovery

---

### 3.1 API Gateway

---

#### 1. What is an API Gateway?

An **API Gateway** is a specialized reverse proxy and Layer-7 (application-level) router that acts as the **single unified entry point** for all external client requests (Web SPAs, iOS/Android apps, third-party partner APIs) entering a microservices architecture.

Instead of exposing dozens of internal microservice URLs and ports directly to clients, all traffic enters through the API Gateway, which evaluates **routes**, applies **predicates and filters** (authentication, rate limiting, logging, transformations), and forwards requests to healthy backend instances.

```text
┌────────────┐
│  Web SPA   ├──┐
└────────────┘  │     ┌───────────────────────────────────────────────┐     ┌──────────────────┐
┌────────────┐  │     │                  API Gateway                  │────►│  Order Service   │
│ Mobile App ├──┼────►│  TLS • JWT Auth • Rate Limit • Route/Filter   │     ├──────────────────┤
└────────────┘  │     │  Trace ID Injection • Request Rewrite         │────►│ Payment Service  │
┌────────────┐  │     └───────────────────────────────────────────────┘     ├──────────────────┤
│ Partner API├──┘                                                           │ Catalog Service  │
└────────────┘                                                              └──────────────────┘
```

##### Common Implementations
- **Java / Spring Ecosystem:** **Spring Cloud Gateway** (non-blocking reactive Netty-based)
- **Cloud-Native / Kubernetes:** **Envoy Gateway**, **Istio Ingress Gateway**, **Kubernetes Gateway API**, **Kong**, **APISIX**, **NGINX Plus**
- **Red Hat / IBM Ecosystem:** **Red Hat 3scale API Management**, **IBM API Connect**
- **Managed Cloud:** **AWS API Gateway**, **Azure API Management**, **Google Apigee**

---

#### 2. Why Do Microservices Need a Gateway?

If clients communicated directly with 30–100 backend microservices without an API Gateway:
1. **Exposed Internal Topology:** Clients would have to track changing internal hostnames, ports, and service splits/merges—making backend refactoring impossible without breaking mobile/web releases.
2. **Duplicated Cross-Cutting Code:** Every single microservice would need to re-implement JWT verification, rate limiting, CORS policies, IP filtering, TLS termination, and access logging identically.
3. **Protocol & Format Mismatch:** Internal services may use **gRPC**, **HTTP/2**, or GraphQL, whereas browsers and external partners expect REST/HTTPS over port `443`.
4. **Chatty Client Round-Trips:** High-latency mobile networks suffer if rendering one home screen requires 12 sequential API calls to 8 different microservices.
5. **Expanded Attack Surface:** Exposing every microservice directly to the public internet increases firewall, certificate, and DDoS management risk.

---

#### 3. Gateway Routing

**Gateway routing** maps incoming external HTTP requests to internal downstream services dynamically—decoupling public API URLs from internal service structure.

- **Path-Based Routing:** `/api/v1/orders/**` → `lb://order-service`, `/api/v1/payments/**` → `lb://payment-service`
- **Host-Based Routing:** `api.example.com` vs `admin.example.com` vs `partners.example.com`
- **Header / Version Routing:** `X-API-Version: 2` → `order-service-v2`
- **Canary / Weighted Traffic Splitting:** Route 95% of traffic to `order-service:v1` and 5% to `order-service:v2`
- **Strangler Fig Migration:** Gradually shift specific legacy monolith paths (`/api/v1/billing/**`) over to newly extracted microservices without changing client URLs.

---

#### 4. Route Predicates

A **Route Predicate** is a boolean condition evaluated against an incoming HTTP request to determine **which configured Route matches** it. Multiple predicates on a single route are combined with logical `AND`.

##### Common Predicate Types (Spring Cloud Gateway / Envoy)
- **`Path`:** `/api/v1/orders/{id}`
- **`Method`:** `GET`, `POST`
- **`Header`:** `X-Request-Id` matching regex `\d+`, or `Authorization` present
- **`Query`:** `?currency=USD`
- **`Host`:** `*.partner.example.com`
- **`Cookie`:** `session_tier=beta`
- **`After` / `Before` / `Between`:** Time-window activation for scheduled releases
- **`Weight`:** Weighted random traffic split for A/B or Canary deployments

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-canary
          uri: lb://order-service-v2
          predicates:
            - Path=/api/v1/orders/**
            - Method=GET,POST
            - Weight=orders-group, 10
          filters:
            - StripPrefix=2
```

---

#### 5. Request Filtering

**Filters** intercept and modify requests and responses in a pipeline (`GatewayFilterChain`) around the upstream service call:

```text
Client ──► [Pre-Filter 1: Auth] ──► [Pre-Filter 2: RateLimit] ──► [Pre-Filter 3: Rewrite]
                                                                          │
                                                                          ▼
Client ◄── [Post-Filter 2: Metrics] ◄── [Post-Filter 1: Sec Headers] ◄── Upstream Service
```

- **Pre-Filters (Before forwarding upstream):**
  - Authenticate & validate JWT / API key
  - Enforce rate limits & quotas
  - Generate `X-Correlation-ID` / W3C `traceparent` headers
  - Rewrite path or strip/add headers
  - Reject invalid requests early (saving downstream service CPU!)
- **Post-Filters (After receiving response from upstream):**
  - Inject security & CORS headers (`Strict-Transport-Security`, `X-Content-Type-Options`)
  - Scrub internal error/server headers (`Server`, `X-Powered-By`)
  - Record latency/status metrics (`Prometheus` timers) and audit logs
- **Global Filters vs Route Filters:** Global filters apply to **all** routes (e.g., tracing, auth, global access logging); Route filters apply to specific routes (e.g., `RewritePath`, `CircuitBreaker`, `AddRequestHeader`).

---

#### 6. Authentication and Authorization

The API Gateway acts as the **Policy Enforcement Point (PEP)** at the cluster boundary:

##### Edge Authentication Flow
1. Client sends `Authorization: Bearer <JWT>` (or opaque session cookie / API Key).
2. Gateway validates the JWT **locally in memory** using cached public **JWKS** keys from the Identity Provider (**Keycloak**, Auth0, Okta): checks cryptographic signature, `exp` (expiration), `iss` (issuer), and `aud` (audience).
3. **Token Translation (Phantom Token Pattern):** If external clients use opaque tokens/cookies, the Gateway introspects/exchanges them at the edge and forwards a signed internal **JWT** downstream so internal services stay stateless.
4. **Preventing Header Spoofing:** Gateway strips any client-supplied `X-User-Id` or `X-User-Roles` headers and injects trusted verified identity claims.

##### Coarse-Grained vs Fine-Grained Authorization (Defense in Depth)
- **At the Gateway (Coarse-grained):** Check route-level scopes/roles (e.g., `/admin/**` requires `ROLE_ADMIN`, `/orders/**` requires `orders:write`).
- **Inside the Microservice (Fine-grained / Resource-level):** Never rely solely on the Gateway! The microservice itself must enforce ownership & domain rules (e.g., *"Can User `u-42` cancel Order `ord-778`? Only if `order.ownerId == u-42` and status is `PENDING`"*).

---

#### 7. Rate Limiting

**Rate Limiting** protects downstream microservices and databases from traffic spikes, abusive bots, DDoS, and noisy tenants by capping how many requests a client may make within a time window.

##### Standard Response Behavior
- Allow request → optionally return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
- Reject excess request → return **`429 Too Many Requests`** with **`Retry-After`** header **before** hitting backend services.

##### Local vs Distributed Rate Limiting
- **Local (In-Memory per Gateway Pod):** Fast (0 network hops), but inaccurate when scaled to 10 Gateway pods (a limit of 100 req/min per pod becomes 1,000 req/min globally).
- **Distributed (Redis + Atomic Lua Script):** All Gateway pods share centralized **Token Bucket** or **Sliding Window** counters in Redis (used by Spring Cloud Gateway `RequestRateLimiter`).
  - **Key Resolver Strategies:** Rate limit by `userId` (from JWT), `tenantId` (multi-tenant SaaS tier), `X-API-Key`, or client IP (`X-Forwarded-For`—careful with corporate NAT!).
  - **Resilience Rule:** Decide **Fail-Open** (allow traffic if Redis goes down so Redis doesn't cause a total outage) vs **Fail-Closed** (for billing/login brute-force endpoints).

---

#### 8. Throttling

Although often used loosely together, **Rate Limiting** and **Throttling** differ in how they handle excess traffic:

| Aspect | Rate Limiting | Throttling (Traffic Shaping) |
| --- | --- | --- |
| **Action on Excess** | **Hard rejection** (`429 Too Many Requests`) immediately | **Slows down / delays / smooths** traffic or caps concurrency |
| **Algorithm** | Token Bucket, Fixed/Sliding Window | **Leaky Bucket** (drains queue at steady rate), **Concurrency Limits**, Priority Shedding |
| **Goal** | Enforce fair usage quotas & protect against abuse | Smooth bursty traffic so downstream DBs/legacy systems process at a steady, sustainable rate |

- **Example:** Tiered throttling lets Gold-tier tenants burst up to 500 RPS while queuing or capping Free-tier bulk exports to 10 concurrent requests.

---

#### 9. Request Transformation

The API Gateway adapts external client expectations to internal service implementations:
- **Path Rewriting:** `GET /api/v1/users/profile` → `RewritePath` → `GET /profile` on `user-service`.
- **Header Manipulation:** Strip sensitive cookies/headers before calling internal services; inject `X-Tenant-Id`, `X-User-Id`, and `X-Forwarded-Proto`.
- **Payload & Protocol Translation:** Accept REST/JSON from browsers and transcode to **gRPC/Protobuf** (Envoy gRPC-JSON transcoder) or SOAP/XML for legacy mainframes.
- **Query Parameter Mapping:** Normalize legacy query parameter names when migrating clients across API versions.

---

#### 10. Response Aggregation

Some API Gateways (or dedicated **BFF** layers behind the gateway) combine responses from multiple microservices (`OrderService` + `CustomerService` + `TrackingService`) into one unified response to reduce mobile network round-trips.

> **Architectural Warning (Avoid "Smart Gateway" Anti-Pattern):**
> Keep heavy business logic, multi-step transactions, and complex response aggregation **out of the shared edge API Gateway**!
> - Putting domain composition in the main Gateway creates coupling, slows Gateway deployments, and risks blocking its event loop.
> - **Best Practice:** Keep the **API Gateway thin and generic** (routing, auth, rate limiting, TLS), and place UI-specific aggregation inside **Backend for Frontend (BFF)** services behind it.

---

#### 11. Load Balancing

When the Gateway resolves a route like `lb://order-service`, it distributes incoming requests across all **healthy** backend instances:
- Integrates with **Service Discovery** (Kubernetes Endpoints, Eureka, Consul) to maintain an up-to-date list of backend Pod IPs.
- Performs **active health checks** and **passive outlier detection** (temporarily ejecting an instance that returns consecutive 5xx errors or timeouts).
- Applies algorithms such as **Round Robin**, **Least Connections**, **Weighted Round Robin** (for canary deployments), or **Consistent Hashing** (for cache affinity).

---

#### 12. Gateway Security

Because the API Gateway sits at the untrusted public perimeter, it enforces layered perimeter security:
1. **TLS/SSL Termination:** Terminates HTTPS (`TLS 1.2 / 1.3`), enforces HSTS, and optionally re-encrypts via **mTLS** to internal services.
2. **CORS Enforcement:** Centralizes `Access-Control-Allow-Origin` and preflight `OPTIONS` handling cleanly in one place.
3. **Request Size & Timeout Guards:** Caps max body size (`413 Payload Too Large`), header length, and request timeouts to defeat Slowloris and memory exhaustion attacks.
4. **WAF Integration:** Blocks common SQL injection, XSS, and path-traversal signatures at the edge.
5. **Information Hiding:** Scrubs internal stack traces, server banners, and internal microservice IPs from error responses.

---

#### 13. Gateway High Availability

Because **100% of external traffic** flows through the API Gateway, it must be architected for zero downtime:
- **Stateless Horizontal Scaling:** Run multiple replicas (`>= 3`) across **multiple Availability Zones (AZs)** behind a redundant Cloud L4 Network Load Balancer / Anycast DNS, with Kubernetes **HPA** autoscaling on CPU/RPS.
- **Externalized State:** Keep routing configs in GitOps / dynamic config stores and rate-limit/session counters in a **highly available Redis Cluster / Sentinel** setup.
- **Graceful Shutdown & PodDisruptionBudget (PDB):** Ensure at least $N-1$ pods stay up during K8s node drains and in-flight requests drain cleanly on `SIGTERM`.

---

#### 14. Gateway as a Single Point of Failure

If the shared API Gateway goes down—or slows down—**every microservice behind it becomes unreachable** to external users.

##### Common Failure Modes & Mitigations
| Risk | Mitigation |
| --- | --- |
| **Thread / Event-Loop Blocking** | Use strictly **non-blocking reactive gateways** (Spring Cloud Gateway on Netty, Envoy in C++). **Never** make blocking JDBC or synchronous HTTP calls inside custom Gateway filters! |
| **Redis Outage Breaks Rate Limiter** | Configure **Fail-Open** timeouts (`50ms` Redis timeout + fallback to local in-memory limiter or pass-through) so Redis hiccups don't blackhole all APIs. |
| **Slow Downstream Exhausts Gateway** | Configure **per-route timeouts**, **Circuit Breakers** (Resilience4j), and **connection pool / bulkhead limits** so one hanging service (`ReportService`) cannot starve Gateway connections for `CheckoutService`. |
| **Bad Config Pushes Crash All Routes** | Validate route configs in CI, roll out via **Canary Gateway** stages, and use **multiple segmented Gateways** (e.g., separate Public Mobile Gateway, Partner B2B Gateway, and Internal Admin Gateway) to shrink blast radius. |

---

#### 15. API Gateway versus Reverse Proxy

| Dimension | Traditional Reverse Proxy (NGINX / HAProxy) | API Gateway (Spring Cloud Gateway / Kong / Envoy / 3scale) |
| --- | --- | --- |
| **Primary Purpose** | Network routing, load balancing, TLS termination, static caching, compression | **API lifecycle & cross-cutting microservice edge logic** |
| **Awareness** | L4/L7 HTTP path, IP, and connection focused | **API-aware:** understands JWTs, OAuth2 scopes, tenant plans, OpenAPI contracts |
| **Security & Auth** | Basic Auth, IP ACLs, mutual TLS | **OAuth2 / OIDC / JWT validation**, JWKS rotation, API keys, fine-grained claims injection |
| **Traffic Governance** | Basic IP connection/request rate limits | **Per-user / per-tenant / per-API-key quotas**, tiered monetization, spike arrest |
| **Transformations** | Header/URL rewrites | Header/body transformations, **REST-to-gRPC transcoding**, response shaping |
| **Service Discovery** | Mostly static upstreams (or dynamic reload) | Native real-time integration with **Kubernetes, Eureka, Consul, Envoy xDS** |

---

#### 16. API Gateway versus Service Mesh

These two technologies are **complementary, not mutually exclusive**:
- **API Gateway** manages **North–South Traffic** (external clients → edge → inside cluster).
- **Service Mesh** (**Istio**, **Linkerd**, **Consul Connect**) manages **East–West Traffic** (microservice-to-microservice communication *inside* the cluster) via **Sidecar Proxies** (e.g., Envoy) injected into every Pod.

```text
                       [ NORTH–SOUTH TRAFFIC ]
  External Client ──► API Gateway (OAuth2, Rate Limit, Public Routing, WAF)
                           │
                           ▼
  ┌─────────────────── Kubernetes Cluster ───────────────────┐
  │                 [ EAST–WEST TRAFFIC ]                    │
  │                                                          │
  │  ┌─────────────────────┐        ┌─────────────────────┐  │
  │  │ [Order Svc | Envoy] ┼─ mTLS ►│[Payment Svc | Envoy]│  │
  │  └──────────┬──────────┘        └─────────────────────┘  │
  │             │ mTLS + Retries + Traces                    │
  │             ▼                                            │
  │  ┌─────────────────────┐                                 │
  │  │[Inventory | Envoy]  │                                 │
  │  └─────────────────────┘                                 │
  └──────────────────────────────────────────────────────────┘
```

| Dimension | API Gateway | Service Mesh (Istio / Linkerd) |
| --- | --- | --- |
| **Traffic Direction** | **North–South** (Edge / Ingress into system) | **East–West** (Internal service-to-service) |
| **Deployment Topology** | Centralized edge cluster (shared gateway pods) | Decentralized **Sidecar Proxy** alongside every service container |
| **Primary Focus** | External auth (OAuth2/OIDC), API keys, per-tenant rate limits, public API versioning, request transformation | Transparent **mTLS** everywhere, zero-trust authZ policies, retries/timeouts/circuit breaking, L7 traffic shifting (canary), automatic distributed metrics/tracing |
| **Application Awareness** | High (public API products, user sessions, client differences) | Infrastructure-transparent (services don't write mesh logic) |

---

### 3.2 Service Discovery

---

#### 1. What is Service Discovery?

**Service Discovery** is the mechanism that enables microservices and gateways to **automatically locate the current network addresses (`IP:Port`) of healthy service instances** at runtime using logical service names (e.g., `payment-service`) instead of hardcoded hostnames or IP addresses.

---

#### 2. Why is Service Discovery Necessary?

In traditional monolithic deployments, services ran on static VMs with fixed IPs (`10.0.1.15:8080`). In modern cloud-native environments (Kubernetes, ECS, VM autoscaling groups), network locations are **ephemeral and constantly changing**:
- **Autoscaling** spins up 20 new pods during flash sales and terminates them afterward.
- **Rolling deployments** continuously replace old pod IPs with new pod IPs.
- **Self-healing / Node failures** reschedule crashed containers onto different host nodes with brand-new IPs.

Manually updating configuration files for ephemeral IPs is impossible—services must register and discover each other **dynamically**.

---

#### 3. Client-Side Discovery

In **Client-Side Discovery**, the **calling microservice** is responsible for querying the **Service Registry**, caching the list of available instances locally, choosing one using an embedded load-balancing library, and making the direct call.

```text
┌───────────────────┐  1. Query instances   ┌──────────────────┐
│   Order Service   ├──────────────────────►│ Service Registry │
│                   │◄──────────────────────┤ (Eureka / Consul)│
│ [LoadBalancerLib] │  2. [10.1.0.5, .6]    └────────▲─────────┘
└────────┬──────────┘                                │ Register
         │ 3. Pick 10.1.0.5 directly                 │
         ▼                                  ┌────────┴─────────┐
┌──────────────────┐                        │ Payment Service  │
│ Payment Pod (.5) │                        │  (.5, .6, .7)    │
└──────────────────┘                        └──────────────────┘
```

- **Examples:** Spring Cloud Netflix Eureka + Spring Cloud LoadBalancer, gRPC client-side load balancing with headless DNS/xDS.
- **Pros:** Fewer network hops (no extra load balancer bottleneck), caller has full control over load-balancing strategies (zone-aware, sticky, weighted).
- **Cons:** Couples every microservice codebase to a specific discovery client SDK and language ecosystem (harder in polyglot Java + Go + Python stacks).

---

#### 4. Server-Side Discovery

In **Server-Side Discovery**, the calling microservice simply calls a single **well-known logical DNS name or Virtual IP** (Router / Load Balancer / Kubernetes `Service`), and the **infrastructure router** queries the Service Registry and forwards the request to a healthy instance.

```text
┌───────────────────┐  1. Call http://payment-svc  ┌─────────────────────────┐
│   Order Service   ├─────────────────────────────►│   Router / LB / K8s     │
│ (Unaware of IPs!) │                              │ (kube-proxy / ALB/Envoy)│
└───────────────────┘                              └──────┬──────────┬───────┘
                                                          │          │ 2. Lookup
                                             3. Forward   │          ▼
                                                          │  ┌────────────────┐
                                                          │  │Service Registry│
                                                          ▼  │(K8s API/Consul)│
                                                  ┌──────────────────┐────────┘
                                                  │ Payment Pod (.5) │
                                                  └──────────────────┘
```

- **Examples:** **Kubernetes `Service` (`ClusterIP` + `kube-proxy` / eBPF)**, AWS Application Load Balancer (ALB), Consul + NGINX/Envoy.
- **Pros:** **Language-agnostic**—application code has zero discovery SDK dependencies and just makes standard HTTP/gRPC calls to `http://payment-service`.
- **Cons:** Adds an infrastructure hop (though Kubernetes `kube-proxy` IPVS/eBPF and Envoy sidecars keep this hop local on the same host node!).

##### Client-Side vs Server-Side Discovery
| Dimension | Client-Side Discovery | Server-Side Discovery |
| --- | --- | --- |
| **Who queries registry?** | The calling application (`EurekaClient`) | Infrastructure router (`kube-proxy`, ALB, Envoy) |
| **Application coupling** | Coupled to discovery/LB library per language | **Completely decoupled** (polyglot-friendly) |
| **Network hops** | Direct caller-to-callee (1 hop) | Caller → Router → Callee (or node-local eBPF/iptables) |
| **Typical Ecosystem** | Legacy Spring Cloud / Eureka on VMs | **Kubernetes**, Cloud LBs, Service Mesh |

---

#### 5. Service Registry

The **Service Registry** is the central, highly available directory storing live metadata for all running microservice instances:
- **Stored Data:** `serviceName → [{instanceId, IP, port, zone/AZ, version, healthStatus, metadata}]`
- **Examples:** Kubernetes Control Plane (`etcd` + `EndpointSlices`), Netflix Eureka, HashiCorp Consul, Apache ZooKeeper.
- **Key Architectural Requirement:** The registry itself must be **clustered across multiple nodes/AZs**—if the registry becomes unavailable and caches expire, services lose the ability to discover new instances.

---

#### 6. Service Registration

How do instances enter the Service Registry?

1. **Self-Registration Pattern:**
   - The microservice itself contacts the registry on startup to register its IP/port and sends periodic heartbeats to renew its lease (e.g., **Eureka Client**).
   - *Drawback:* Couples service code to the registry and assumes the service is healthy enough to manage its own registration (a hung JVM can sometimes still answer heartbeat timers while failing business requests!).
2. **Third-Party Registration Pattern (Registrar):**
   - An independent infrastructure component (**Registrar**) monitors the lifecycle of containers/VMs, polls health probes, and registers/deregisters instances on their behalf.
   - *Example:* **Kubernetes Control Plane** (`kubelet` runs `readinessProbe` → updates `Pod` condition → `EndpointSlice` controller adds/removes the Pod IP automatically).

---

#### 7. Service Deregistration

Removing an instance promptly when it stops prevents callers from getting connection errors:

1. **Graceful Deregistration (Planned Shutdown / Rolling Update):**
   - Orchestrator sends `SIGTERM` to the pod.
   - Service stops accepting new work (marks readiness probe `DOWN` / sends `DELETE` to Eureka), waits a brief propagation window (e.g., `preStop` sleep of 5–15s in K8s so `kube-proxy`/Gateway rules finish updating), drains active in-flight HTTP requests, closes DB/Kafka connections, and exits cleanly (`0`).
2. **Ungraceful Deregistration (Crash / OOMKill / Hardware Loss):**
   - The process dies abruptly without sending a deregister call.
   - The registry detects missed heartbeats or consecutive failed health probes and evicts the dead IP after its **TTL / failure threshold** expires.

---

#### 8. Health Checks and Heartbeats

Health checking determines **which registered instances should receive traffic**:

- **Heartbeats (Push Model):** The service instance periodically sends a lightweight `"I'm alive"` pulse to the registry (e.g., Eureka defaults to every **30s**, evicting after **90s** of missed heartbeats).
- **Active Health Checks (Pull Model):** The registry or orchestrator actively invokes an endpoint (`GET /actuator/health/readiness`) on a schedule (e.g., Kubernetes probes every **5–10s**; failing 3 times removes the pod from `EndpointSlices`).

##### Critical Health Check Rule — Shallow vs Deep Checks
- **Liveness Probe (`/health/liveness`):** Should **only** check if the JVM/process itself is deadlocked or unrecoverable. **Never** fail liveness because a downstream service or database is down (restarting pods won't fix an external DB outage and causes crash loops!).
- **Readiness Probe (`/health/readiness`):** Determines if the pod can serve traffic (e.g., local DB connection pool ready, caches warmed, not shutting down). Avoid failing readiness because a non-critical *downstream microservice* is down—otherwise a single downstream failure marks *all* upstream pods NotReady and causes a **cluster-wide cascading outage**!

---

#### 9. Registry Consistency

Applying the **CAP Theorem** to Service Registries forces an architectural trade-off during network partitions:

| Property | **AP Registry (e.g., Netflix Eureka)** | **CP Registry (e.g., Consul / etcd / ZooKeeper)** |
| --- | --- | --- |
| **Behavior under Partition** | Favors **Availability**: all surviving nodes continue serving cached registrations (even if some might be slightly stale) | Favors **Strong Consistency** (Raft/Zab quorum): minority partition rejects writes/reads rather than serving unverified registrations |
| **Failure Mode** | Callers may occasionally hit a dead IP and must retry on another instance | During leader election or split-brain without quorum, **new services cannot register** and discovery may fail |
| **Why AP Often Fits Discovery** | In microservices, callers *already* need retries/timeouts for network glitches; slightly stale discovery with retry beats 100% discovery outage | Best when strict single-leader coordination or distributed locking (`Leader Election`) is required alongside discovery |

---

#### 10. Stale Service Registrations

**Stale registrations** occur when a service instance has crashed or started shutting down, yet callers still route traffic to its old IP because of **multi-layer caching & propagation delays**:
- Heartbeat / probe detection delay (e.g., 10–90s)
- Client local registry cache refresh interval (e.g., Eureka 30s)
- JVM / OS DNS TTL caching
- Asynchronous `iptables`/`kube-proxy` update delay across Kubernetes nodes during `Pod` termination

##### Mitigations
1. **Kubernetes `preStop` Hook + Graceful Shutdown:** Wait 5–10 seconds after `SIGTERM` (while `readiness` is already removed) before closing the HTTP listener so late packets still arriving from stale caches succeed instead of throwing `Connection Refused`.
2. **Fast Connect Timeouts (`500ms`) + Safe Retries:** If a client hits a dead IP and gets `ConnectException` (connection never established!), immediately retry on the next healthy instance in the list.
3. **Passive Outlier Detection (Circuit Breaking at LB):** Envoy / Resilience4j immediately ejects an instance from the local LB pool after $N$ consecutive connection errors or 5xx responses without waiting for registry eviction.

---

#### 11. Dynamic Service Discovery

Modern discovery avoids slow 30–60 second polling intervals in favor of **event-driven watch streams**:
- **Kubernetes API Watches (`EndpointSlices`):** `kube-proxy`, Ingress controllers, and Envoy xDS control planes hold open streaming watches on `EndpointSlices`, receiving pod IP additions/removals within **milliseconds to ~1 second**.
- **Envoy xDS (`EDS` — Endpoint Discovery Service):** Control planes (Istio `istiod`) push real-time endpoint updates over gRPC streams directly to every Envoy proxy.
- **Consul Blocking Queries:** Clients issue HTTP long-polling requests that return immediately whenever the service index changes.

---

#### 12. Service Discovery Using Eureka

**Netflix Eureka** (part of Spring Cloud Netflix) is an **AP (Eventually Consistent), Client-Side, Self-Registration** service discovery system widely used in Spring Boot VM/non-K8s deployments:
- **Eureka Server:** Clustered registry peers replicating registrations asynchronously between each other.
- **Eureka Client (`@EnableDiscoveryClient`):** Registers on boot, sends heartbeats every **30s**, and fetches registry updates every **30s** (which means up to ~90–120s worst-case propagation lag!).
- **Self-Preservation Mode:** If Eureka Server suddenly stops receiving heartbeats from **>15% of registered instances** within 1 minute, it assumes a **network partition** occurred between Eureka and the services (rather than all instances dying simultaneously) and **freezes evictions** to protect availability.
- **Modern Note:** When deploying Spring Boot on **Kubernetes**, disable Eureka and use native **Kubernetes Service Discovery** (avoid running two redundant discovery registries).

---

#### 13. Consul Fundamentals

**HashiCorp Consul** is a multi-platform, **CP (Strongly Consistent via Raft)** service registry, health checker, distributed KV store, and service mesh control plane:
- **Architecture:** A cluster of 3 or 5 **Consul Servers** running Raft consensus, plus a lightweight **Consul Client Agent** running on every host/node.
- **Registration & Health Checks:** Services register with their local node agent, which executes active **HTTP, TCP, gRPC, or script health checks** locally (reducing load on central servers).
- **Discovery Interfaces:** Services can query Consul via **DNS** (`payment-service.service.consul`) *or* **HTTP API** (`/v1/health/service/payment-service?passing=true`), supporting both server-side and client-side discovery.
- **Consul Connect:** Automates service-to-service **mTLS** with identity-based intentions and Envoy sidecars.

---

#### 14. Kubernetes Service Discovery

Kubernetes provides built-in, zero-code **Server-Side & Third-Party Service Discovery**:

```text
1. Pod passes ReadinessProbe ──► K8s Controller adds Pod IP to EndpointSlice
2. CoreDNS resolves "payment-service.prod.svc.cluster.local" ──► ClusterIP (10.96.45.12)
3. Order Pod calls http://payment-service:8080
4. Node kube-proxy (iptables / IPVS / Cilium eBPF) DNATs 10.96.45.12 ──► Healthy Pod IP (10.244.2.18:8080)
```

##### Key Kubernetes Service Discovery Mechanisms
- **`ClusterIP` (Default):** Stable virtual IP inside the cluster; `kube-proxy` load balances L4 TCP connections across healthy backend Pod IPs in `EndpointSlice`.
- **Cluster DNS Names:**
  - Same namespace: `http://payment-service`
  - Cross-namespace FQDN: `http://payment-service.prod.svc.cluster.local`
- **Headless Service (`clusterIP: None`):** CoreDNS returns **all individual healthy Pod IPs** (`A`/`AAAA` DNS records) instead of one Virtual IP. Essential for:
  - **gRPC / HTTP/2 client-side load balancing** (prevents HTTP/2 single-connection pinning on `ClusterIP`!).
  - **StatefulSets** (Kafka brokers, databases needing direct pod addresses `kafka-0.kafka-headless`).
- **Spring Cloud Kubernetes:** Allows Spring Boot apps to query `DiscoveryClient` directly against the K8s API (`EndpointSlices`) if client-side load balancing or metadata filtering is desired.

---

#### 15. Load-Balancing Algorithms

Once healthy instances are discovered, the load balancer (Gateway, Envoy, `kube-proxy`, or Spring Cloud LoadBalancer) picks a target instance using one of these core algorithms:

##### A. Round Robin
- **How it works:** Cycles sequentially through the instance list: `Instance 1 → Instance 2 → Instance 3 → Instance 1`.
- **Pros:** Extremely simple, $O(1)$, guarantees equal request count per instance over time.
- **Cons:** Assumes all instances have identical hardware and all requests have equal execution cost (a slow query on Pod 1 still receives the same new requests as idle Pod 2).

##### B. Weighted Round Robin
- **How it works:** Assigns a numeric weight to each instance/version; instances with higher weights receive proportionally more turns (`v1: weight 90`, `v2: weight 10`).
- **Best For:**
  - **Canary releases** (sending 5–10% of production traffic to a new version).
  - Heterogeneous hardware (sending 2x traffic to a 16-vCPU node vs an 8-vCPU node).

##### C. Least Connections (or Least Requests)
- **How it works:** Routes the incoming request to the instance that currently has the **fewest active in-flight connections/requests**.
- **Pros:** **Self-adapting to latency skew!** If Pod 2 experiences a GC pause or slow DB lock and active requests pile up, traffic automatically shifts to faster Pods 1 and 3.
- **Cons:** Requires tracking live active request counts per backend (higher overhead in distributed uncoordinated load balancers; easily handled by Envoy/NGINX/HAProxy).

##### D. Random Selection (and Power of Two Random Choices — P2C)
- **Random Selection:** Picks an instance uniformly at random (`ThreadLocalRandom.nextInt(n)`).
  - *Pros:* Zero shared state, avoids synchronization contention, prevents multiple uncoordinated load balancers from hitting the exact same pod in lockstep ("herd behavior" seen when 10 gateways all start Round Robin at index 0).
  - *Cons:* Short-term variance can temporarily overload one instance.
- **Power of Two Random Choices (P2C / Least-of-Two):** Picks **2 random instances** and routes to whichever of the two has **fewer active connections** (or lower latency).
  - Used by **Envoy**, **Linkerd**, and **NGINX** because it achieves nearly optimal Least-Connections load balancing with $O(1)$ scalability and zero global coordination!

##### Load-Balancing Algorithms Comparison
| Algorithm | State Needed | Handles Slow Pods? | Best Use Case |
| --- | --- | --- | --- |
| **Round Robin** | Counter | No | Homogeneous pods with fast, uniform request times |
| **Weighted Round Robin** | Counter + Weights | No (static weights) | Canary deployments, mixed-capacity instances |
| **Least Connections** | Active connection count | **Yes** | Variable request durations (reports, DB queries, streaming) |
| **Random / P2C** | None (Random) / 2 local reads (P2C) | **Yes (with P2C)** | Large fleets with multiple uncoordinated load balancers / sidecars |

---

#### Quick Interview Revision Checklist (Section 3)
- Can you explain **why microservices need an API Gateway** and list what belongs in a Gateway vs what belongs in a **BFF** or **Service Mesh**?
- Can you trace how **Route Predicates** and **Pre/Post Filters** execute, and how **edge JWT auth + downstream fine-grained authZ** work together?
- Can you differentiate **Rate Limiting** vs **Throttling** and explain how distributed rate limiting works with **Redis + Lua** (and Fail-Open vs Fail-Closed)?
- Can you explain how to prevent the **API Gateway from becoming a Single Point of Failure** (non-blocking I/O, per-route circuit breakers/timeouts, segmented gateways, multi-AZ)?
- Can you compare **Client-Side vs Server-Side Discovery** and **Self-Registration vs Third-Party Registration**?
- Can you explain **AP (Eureka + Self-Preservation)** vs **CP (Consul + Raft)** registry consistency and how to mitigate **stale registrations** (`preStop` sleep, fast connect timeout + retry, outlier detection)?
- Can you walk through **Kubernetes Service Discovery** end-to-end (`ReadinessProbe → EndpointSlice → CoreDNS → ClusterIP / Headless Service → kube-proxy/eBPF`)?
- Can you explain when to choose **Round Robin**, **Weighted Round Robin**, **Least Connections**, and **Power of Two Random Choices (P2C)**?
