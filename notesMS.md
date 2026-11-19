# Microservices Study Notes

## 1. Microservices Fundamentals

### 1.1 Architecture Fundamentals

---

#### 1. What are Microservices?

**Microservices architecture** is an architectural style that structures an application as a collection of small, autonomous, loosely coupled, and independently deployable services modeled around specific **business capabilities** (bounded contexts).

Each microservice:
- Owns a single cohesive business domain (e.g., `OrderService`, `PaymentService`, `InventoryService`).
- Runs in its own OS process / container and exposes well-defined APIs or asynchronous event contracts.
- Owns its private data store (**Database per Service**)—no direct database sharing or cross-service SQL joins.
- Communicates over lightweight mechanisms such as **REST/HTTP**, **gRPC**, or asynchronous messaging (**Kafka**, **RabbitMQ**).
- Can be developed, tested, deployed, and scaled independently by a small, cross-functional team.

```text
                             ┌──────────────────┐
                        ┌───►│  Order Service   ├───► [( Order DB )]
                        │    └────────┬─────────┘
                        │             │ Events (Kafka)
┌────────┐   ┌──────────┴──┐          ├─────────────────────────────────┐
│ Client ├──►│ API Gateway │          ▼                                 │
└────────┘   └──────────┬──┘ ┌──────────────────┐                       │
                        ├───►│ Payment Service  ├───► [( Payment DB )]  │
                        │    └──────────────────┘                       │
                        │    ┌──────────────────┐                       │
                        └───►│Inventory Service │◄──────────────────────┘
                             └────────┬─────────┘
                                      ▼
                             [( Inventory DB )]
```

> **Interview Takeaway:** Microservices are not defined by "lines of code" or "tiny size"—they are defined by **independent deployability**, **bounded business ownership**, and **data isolation**.

---

#### 2. Monolithic Architecture

A **monolithic architecture** packages the entire application—UI/API controllers, business logic across all domains, background jobs, and data access layers—into a **single deployable unit** (e.g., one `.jar`, `.war`, or single container image) typically backed by a **single shared relational database**.

```text
┌────────┐   ┌───────────────┐   ┌──────────────────────────────────────────┐
│ Client ├──►│ Load Balancer ├──►│     Single Deployable Unit (app.jar)     │
└────────┘   └───────────────┘   │  ┌────────────────────────────────────┐  │
                                 │  │        REST Controllers / UI       │  │
                                 │  └──────┬──────────┬───────────┬──────┘  │
                                 │         ▼          ▼           ▼         │
                                 │   ┌──────────┐ ┌────────┐ ┌───────────┐  │
                                 │   │  Orders  │ │Payments│ │ Inventory │  │
                                 │   │  Module  │ │ Module │ │  Module   │  │
                                 │   └────┬─────┘ └───┬────┘ └─────┬─────┘  │
                                 └────────┼───────────┼────────────┼────────┘
                                          └───────────┼────────────┘
                                                      ▼
                                         ┌─────────────────────────┐
                                         │ Single Shared Database  │
                                         └─────────────────────────┘
```

##### Strengths
- **Simple development & debugging:** Single codebase, single IDE run configuration, straightforward stack traces.
- **Strong ACID consistency:** Cross-domain updates (e.g., creating an order and deducting inventory) happen inside one local database transaction (`@Transactional`).
- **Low inter-module latency:** Modules communicate via in-memory method calls (nanoseconds) rather than network hops (milliseconds).
- **Simple deployment & operations:** Only one artifact and one database to deploy, monitor, and back up.

##### Limitations
- **Tight coupling over time:** Without strict boundaries, modules inevitably reach into each other's internal classes and database tables ("Big Ball of Mud").
- **Coarse-grained scaling:** If only the search endpoint gets 10x traffic, you must scale the entire monolith (wasting CPU/RAM on unused modules).
- **Large blast radius:** A memory leak, thread-pool exhaustion, or uncaught fatal error in one minor module can crash the entire process.
- **Slow build/deploy feedback loops:** Any 1-line change forces rebuilding, retesting, and redeploying the whole application.
- **Technology lock-in:** Hard to adopt a new language, runtime, or framework version incrementally.

---

#### 3. Modular Monolith

A **modular monolith** is a **single deployable unit** that enforces **strict internal modular boundaries** by business domain. Each module exposes only explicit public interfaces (or domain events) and owns its logical database schema or dedicated tables without cross-module SQL joins or direct access to another module's internal repositories.

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                   Single Deployable Artifact (app.jar)                     │
│                                                                            │
│  ┌─────────────────────┐   ┌─────────────────────┐   ┌──────────────────┐  │
│  │    Order Module     │   │   Payment Module    │   │ Inventory Module │  │
│  │  ┌───────────────┐  │   │  ┌───────────────┐  │   │ ┌──────────────┐ │  │
│  │  │ Public API /  ├──┼──►│  │ Public API /  │  │   │ │ Public API / │ │  │
│  │  │    Events     ├──┼───┼──┴───────────────┴──┼──►│ │    Events    │ │  │
│  │  └───────┬───────┘  │   │  └───────┬───────┘  │   │ └──────┬───────┘ │  │
│  │          ▼          │   │          ▼          │   │        ▼         │  │
│  │  [Internal Logic]   │   │  [Internal Logic]   │   │ [Internal Logic] │  │
│  └──────────┬──────────┘   └──────────┬──────────┘   └────────┬─────────┘  │
└─────────────┼─────────────────────────┼───────────────────────┼────────────┘
              ▼                         ▼                       ▼
     [( orders_schema )]      [( payments_schema )]   [( inventory_schema )]
            └───────────────────────────┼───────────────────────────┘
                         Single DB Server (Isolated Schemas)
```

##### How It Works in Java
- **Spring Modulith:** Structures packages as application modules, verifies dependency rules at test time, supports inter-module communication via Spring application events (`@ApplicationModuleListener`), and even provides an event publication registry (transactional outbox behavior inside one DB).
- **ArchUnit / Java Platform Module System (JPMS):** Enforces compile-time or test-time constraints so `com.app.order` cannot directly import `com.app.payment.internal.*` or query `payment_tables`.

##### Why Teams Choose a Modular Monolith ("Monolith First")
- Delivers clean domain boundaries, parallel team ownership, and easy refactoring **without** network latency, distributed transactions, service discovery, or Kubernetes operational overhead.
- If a module later needs independent scaling or independent deployment (e.g., `VideoProcessingModule`), extracting it into a microservice is straightforward because its API and schema boundaries are already clean.

---

#### 4. Service-Oriented Architecture (SOA)

**Service-Oriented Architecture (SOA)** is an enterprise architectural style developed to integrate large, heterogeneous enterprise systems (such as ERP, CRM, billing, and legacy mainframes) by exposing reusable, coarse-grained services connected through a central **Enterprise Service Bus (ESB)**.

```text
┌────────────┐        ┌───────────────────────────────────────────┐
│ Web Portal ├───────►│     Enterprise Service Bus (Smart Pipe)   │
└────────────┘        │  • Routing & Orchestration                │
┌────────────────┐    │  • Protocol & XML/JSON Transformation     │
│ Partner System ├───►│  • Central Business Rules                 │
└────────────────┘    └───────┬─────────────┬──────────────┬──────┘
                              │             │              │
                              ▼             ▼              ▼
                      ┌──────────────┐ ┌──────────┐ ┌─────────────┐
                      │   Billing    │ │   CRM    │ │   Legacy    │
                      │Service (SOAP)│ │Subsystem │ │Mainframe(MQ)│
                      └──────┬───────┘ └────┬─────┘ └─────────────┘
                             └───────┬──────┘
                                     ▼
                       [( Shared Enterprise Database )]
```

##### Key Characteristics of Traditional SOA
- **Smart pipes, dumb endpoints:** The ESB performs heavy lifting—message routing, XML/XSLT transformation, schema validation, orchestration (BPEL), and sometimes business logic.
- **Heavyweight protocols & standards:** Typically relies on **SOAP**, **WSDL**, **XML**, **WS-Security / WS-AtomicTransaction**, and enterprise message queues (IBM MQ, JMS).
- **Shared data & canonical models:** Services frequently share an enterprise canonical data model and often read/write to a shared enterprise database.
- **Coarse-grained & centralized governance:** Services represent large enterprise subsystems, and releases/contracts are often governed by a central architecture board.

> **Note:** Microservices evolved as a lightweight, cloud-native refinement of SOA principles—keeping service orientation while rejecting the central ESB bottleneck and shared database coupling.

---

#### 5. Microservices versus Monolith

| Dimension | Monolithic Architecture | Microservices Architecture |
| --- | --- | --- |
| **Deployment Unit** | Single deployable artifact (`.jar` / `.war` / image) | Multiple independently deployable services |
| **Codebase & Teams** | Single repository; teams coordinate merges and releases | Separate repos or clear service ownership; small autonomous teams |
| **Data Management** | Single shared database; SQL joins across all tables | Database per service; no cross-service DB access or joins |
| **Inter-Module Communication** | In-process method calls (fast, reliable) | Network calls (REST, gRPC, Kafka/RabbitMQ — higher latency, can fail) |
| **Data Consistency** | Strong ACID local transactions (`@Transactional`) | Eventual consistency (Saga, Outbox, compensating actions) |
| **Scalability** | Must scale the entire application as one unit | Independent horizontal/vertical scaling per service workload |
| **Fault Isolation (Blast Radius)** | Single bug (e.g., OOM, thread leak) can crash entire app | Failure isolated to one service (if timeouts/circuit breakers are used) |
| **Technology Stack** | Single language/framework/runtime version | Polyglot allowed (e.g., Java/Spring Boot, Quarkus, Go, Python) |
| **Testing & Debugging** | Simple local setup and end-to-end debugging | Harder; requires contract tests, Testcontainers, distributed tracing |
| **Operational Complexity** | Low (single pipeline, single log stream, simple monitoring) | High (CI/CD per service, service discovery, API gateway, K8s, OpenTelemetry) |

---

#### 6. Microservices versus SOA

| Dimension | Service-Oriented Architecture (SOA) | Microservices Architecture |
| --- | --- | --- |
| **Primary Goal** | Enterprise-wide reuse & integrating legacy systems | Agility, independent deployment, cloud-native scalability |
| **Service Granularity** | Coarse-grained (large enterprise subsystems) | Fine-grained (focused single business capability / bounded context) |
| **Communication Philosophy** | **Smart pipes, dumb endpoints** (central ESB routes, transforms, orchestrates) | **Dumb pipes, smart endpoints** (services own logic; simple HTTP/gRPC or broker pipes) |
| **Protocols** | SOAP, XML, WSDL, WS-*, JMS, Enterprise MQ | REST/JSON, HTTP/2, gRPC, GraphQL, Kafka, RabbitMQ |
| **Data Storage** | Frequently shared enterprise databases & canonical data models | Strict **database per service** (shared DB is an anti-pattern) |
| **Deployment** | Often coupled or coordinated enterprise release cycles | Fully independent CI/CD per microservice |
| **Failure Risk at Bus** | ESB can become a central bottleneck and single point of failure | Decentralized; no central orchestration bus required |
| **Governance** | Centralized governance & enterprise schema standards | Decentralized governance; teams pick optimal tech/tools within guardrails |

---

#### 7. Advantages and Disadvantages

##### Advantages
1. **Independent Deployability & Faster Time-to-Market:** Teams ship features and bug fixes for one service without waiting on a company-wide release train.
2. **Independent Scalability & Cost Efficiency:** Scale only bottleneck services (e.g., `SearchService` or `CheckoutService`) instead of duplicating the whole system.
3. **Fault Isolation:** A crash or traffic spike in a non-critical service (e.g., `Recommendations`) does not bring down core capabilities (`Checkout`) when resilience patterns are applied.
4. **Technological & Persistence Flexibility:** Teams can pick the best database (PostgreSQL for orders, Redis for caching, Elasticsearch for search) or runtime (Quarkus native image for low memory/fast startup, Spring Boot for rich ecosystem).
5. **Clear Team Ownership (Conway's Law Alignment):** Small "two-pizza" teams own a domain end-to-end, reducing cross-team coordination friction as an organization grows.
6. **Easier Incremental Modernization:** Individual services can be rewritten or upgraded without a risky "big bang" rewrite.

##### Disadvantages
1. **Distributed Systems Complexity:** Every cross-service call crosses an unreliable network, introducing latency, timeouts, retries, partial failures, and serialization overhead.
2. **Data Consistency Overhead:** Replacing local ACID transactions requires **Saga**, **Transactional Outbox**, **Idempotency**, and **Eventual Consistency**, increasing code complexity.
3. **High Operational & Infrastructure Burden:** Requires automated CI/CD, containers, orchestration (Kubernetes), API gateways, service discovery, secrets management, and centralized observability.
4. **Harder Debugging & Troubleshooting:** A single user request may touch 10+ services; diagnosing latency or 500 errors requires distributed tracing (OpenTelemetry) and correlation IDs.
5. **Testing Complexity:** End-to-end tests are slow and brittle; teams must invest in consumer-driven contract testing and Testcontainers.
6. **Risk of Distributed Monolith:** Poorly chosen boundaries create services that must be deployed together and call each other synchronously in deep chains—delivering the worst of both worlds.

---

#### 8. Characteristics of Microservices

Derived from Martin Fowler and James Lewis's foundational definition, mature microservices share these core characteristics:

1. **Componentization via Services:** Components are independent out-of-process services accessed via network interfaces rather than in-memory libraries, enabling independent upgrades and deployments.
2. **Organized Around Business Capabilities:** Teams are cross-functional and domain-aligned (Order team, Billing team) rather than siloed by technical layer (UI team, Middleware team, DBA team).
3. **Products, Not Projects:** Teams follow *"You build it, you run it"*—owning a service across its entire production lifecycle, SLAs, monitoring, and incidents.
4. **Smart Endpoints and Dumb Pipes:** Business logic lives inside services; communication infrastructure (HTTP, Kafka) simply moves requests or events without embedding heavy routing/transformation logic like an ESB.
5. **Decentralized Governance:** Teams can select languages, frameworks, and tools suited to their problem rather than conforming to a single rigid enterprise standard.
6. **Decentralized Data Management:** Each service encapsulates its own database and schema; data sharing happens exclusively via APIs or domain events, never shared tables.
7. **Infrastructure Automation:** Automated build/test/deploy pipelines, containerization, and Infrastructure as Code (IaC) are mandatory to manage tens or hundreds of services reliably.
8. **Design for Failure:** Services assume downstream dependencies and networks *will* fail; timeouts, retries with backoff/jitter, circuit breakers, bulkheads, and fallbacks are first-class architecture concerns.
9. **Evolutionary Design:** Services are designed to be replaced, split, or retired safely over time using versioned contracts and backward compatibility.

---

#### 9. Independent Development and Deployment

**Independent development and deployment** means a team can modify, test, build, release, and roll back a single microservice **without coordinating code changes or simultaneous deployments with any other service**.

##### What Makes It Possible?
- **Strict API & Event Contracts:** Breaking changes are avoided via additive schema evolution, API versioning (`/v1`, `/v2`), and **Expand-and-Contract** database migrations.
- **Consumer-Driven Contract Testing (e.g., Pact, Spring Cloud Contract):** Verifies that provider changes never break consumer expectations before deployment.
- **Dedicated CI/CD Pipelines & Artifacts:** Each microservice builds its own container image and deploys independently (Rolling, Blue-Green, or Canary releases).
- **Decoupled Data Storage:** No shared database tables or foreign keys across services that would force lockstep schema upgrades.

##### Warning Sign — The Distributed Monolith
If deploying `OrderService v2.1` requires simultaneously deploying `InventoryService v3.4` and `CustomerService v1.8` because of shared libraries, shared DB schemas, or breaking synchronous contract changes, **you have a distributed monolith**—all the network cost of microservices with none of the deployment autonomy.

---

#### 10. Independent Scalability

**Independent scalability** allows each microservice to scale compute, memory, and replica count strictly according to its own workload profile.

##### Scaling Dimensions
- **Horizontal Scaling (Scale Out):** Running more replicas behind a load balancer / Kubernetes Service via Horizontal Pod Autoscaler (HPA) based on CPU, memory, RPS, or Kafka consumer lag.
- **Vertical Scaling / Resource Tuning:** Allocating high CPU to compute-heavy services (`FraudScoringService`), high RAM to caching/in-memory services, or native compilation (Quarkus/GraalVM) for fast autoscaling cold starts.

##### Concrete Example (E-Commerce Flash Sale)
- `CatalogReadService`: 80% of traffic, read-heavy → scaled out to **30 pods** + Redis cache.
- `CheckoutService`: moderate traffic, strict consistency → **8 pods**.
- `InvoicePdfService`: background async worker consuming Kafka → scaled based on **Kafka lag**, spikes to **15 pods** post-sale, then scales down to **1 pod**.
- `AdminReportingService`: internal daily batch → stays at **1 pod**.

> **Requirement:** For horizontal scaling to work seamlessly, microservices must remain **stateless** (no local in-memory HTTP sessions or local disk state; push session/shared state to Redis, databases, or object storage).

---

#### 11. Fault Isolation

**Fault isolation** ensures that a failure, resource leak, or traffic surge in one microservice is contained within its own process boundary and does not crash unrelated services.

##### Process-Level Isolation
- In a monolith, an `OutOfMemoryError`, infinite loop, or memory leak in an export feature crashes the entire JVM and takes down login, checkout, and payments.
- In microservices, each service runs in its own container/JVM with dedicated CPU/memory limits; if `RecommendationService` crashes with OOM, `OrderService` and `PaymentService` keep running.

##### Crucial Caveat: Isolation Requires Resilience Patterns
Fault isolation is **not automatic** when services make synchronous calls:
- Suppose `OrderService` calls `RecommendationService` synchronously without timeouts.
- If `RecommendationService` hangs (not crashes, just becomes very slow), `OrderService` request threads block waiting for responses.
- Soon `OrderService`'s thread pool and HTTP connection pool exhaust, causing `OrderService` to fail too (**cascading failure**).

To preserve fault isolation across network boundaries, you must combine:
- **Timeouts** (prevent indefinite thread blocking)
- **Bulkheads** (isolate thread/connection pools per downstream dependency)
- **Circuit Breakers** (fail fast when a dependency is unhealthy, e.g., Resilience4j)
- **Fallbacks / Graceful Degradation** (e.g., return empty/cached recommendations and still complete checkout)
- **Asynchronous decoupling** (use message brokers where real-time response isn't needed)

---

#### 12. Distributed Systems Fundamentals

Once you split a monolith into microservices, your system becomes a **distributed system**, governed by fundamental network and consensus constraints.

##### A. The 8 Fallacies of Distributed Computing (Peter Deutsch)
1. **The network is reliable** → Packets drop, links flap, brokers restart; design with retries, idempotency, and timeouts.
2. **Latency is zero** → In-memory calls take nanoseconds; HTTP/gRPC calls take milliseconds and degrade tail latency (P95/P99) when chained.
3. **Bandwidth is infinite** → Large JSON payloads and chatty APIs saturate NICs and burn CPU on serialization; use pagination, projections, or binary formats (Protobuf/gRPC).
4. **The network is secure** → Zero-trust is required; enforce TLS/mTLS, OAuth2/JWT validation, and service identity even on internal clusters.
5. **Topology doesn't change** → Containers scale, reschedule, and get new IPs constantly; rely on dynamic service discovery (Kubernetes DNS, Consul, Eureka).
6. **There is one administrator** → Multiple teams, cloud providers, and third-party APIs operate independently; standard observability is vital.
7. **Transport cost is zero** → Serialization/deserialization (JSON parsing, TLS handshakes, HTTP connection pooling) consumes real CPU and memory.
8. **The network is homogeneous** → Services run across heterogeneous runtimes, OS kernels, and serialization versions; enforce backward-compatible contracts.

##### B. CAP Theorem
In any distributed data store during a **Network Partition (P)**—which is unavoidable in real networks—the system must choose between:
- **Consistency (C) [Linearizability]:** Every read receives the most recent write or an error.
- **Availability (A):** Every request receives a non-error response, without guaranteeing it contains the most recent write.

Because partitions **will** happen (`P` is non-optional), systems trade off **CP** (reject writes/reads on the minority side, e.g., banking balances, ZooKeeper/etcd) versus **AP** (remain writable/readable on both sides and reconcile later via eventual consistency, e.g., shopping carts, DNS, Dynamo/Cassandra).

##### C. PACELC Theorem (Extension of CAP)
- **If Partition (P):** Trade off **A**vailability vs **C**onsistency.
- **Else (E - normal operation without partition):** Trade off **L**atency vs **C**onsistency (replicating synchronously before acking raises latency; replicating asynchronously keeps latency low at the risk of stale reads).

##### D. Core Network Realities
- **Partial Failures:** Service A succeeds, Service B times out—you cannot tell if B never received the request, crashed mid-flight, or succeeded and the response dropped.
- **No Global Clock:** Clocks drift across nodes (NTP skew); never rely solely on wall-clock timestamps across services for event ordering (use sequence numbers, Kafka partition offsets, or logical clocks).
- **At-Least-Once Delivery & Idempotency:** Retrying a timed-out request can execute it twice unless operations are **idempotent** (via idempotency keys or deduplication).

---

#### 13. When Not to Use Microservices

Microservices solve **organizational scaling**, **independent deployment**, and **workload isolation** problems—if you don't have those problems yet, microservices often slow teams down.

Avoid starting with microservices when:
1. **Early-stage startups / Unclear domain boundaries:** In the discovery phase, business models pivot rapidly. Refactoring domain boundaries inside a monolith takes hours; refactoring boundaries across distributed microservices, APIs, and Kafka topics takes weeks.
2. **Small engineering teams (< 5–10 developers):** Managing 10+ repositories, CI/CD pipelines, Kubernetes clusters, distributed tracing, and Sagas consumes disproportionate capacity that should go into building product features.
3. **Tightly coupled domains needing strict ACID transactions:** If virtually every business workflow requires immediate, strongly consistent cross-entity updates with low latency, splitting into separate databases adds heavy Saga/compensation complexity for little benefit.
4. **Low scale or uniform traffic:** If a 2–4 instance monolith easily handles peak traffic with acceptable deployment speed, distributed overhead provides negative ROI.
5. **Lack of DevOps & Platform maturity:** Without automated CI/CD, container orchestration, centralized logging, metrics, and distributed tracing, microservices become nearly impossible to operate reliably in production.

> **Industry Best Practice ("Monolith First"):** Start with a **Modular Monolith**. Extract specific modules into microservices **only when driven by concrete pain points**—such as independent scaling bottlenecks, conflicting team release cadences, strict compliance isolation, or differing runtime requirements.

---

#### 14. Microservices Architecture Challenges

| Challenge Area | Why It Happens | Typical Solutions / Patterns |
| --- | --- | --- |
| **Distributed Data Consistency** | No cross-service ACID transactions or DB joins | **Saga pattern** (choreography/orchestration), **CQRS**, **Eventual Consistency**, **Transactional Outbox** |
| **Network Unreliability & Cascading Failures** | Synchronous calls fail, hang, or trigger retry storms | **Timeouts**, **Exponential Backoff + Jitter**, **Circuit Breakers** (Resilience4j), **Bulkheads**, **Rate Limiting** |
| **Duplicate Messages & Partial Failures** | Network timeouts cause retries; brokers deliver at-least-once | **Idempotency Keys**, **Transactional Inbox / Deduplication**, **Dead-Letter Queues (DLQ)** |
| **Observability & Debugging** | Requests hop across many async & sync services | **Correlation/Trace IDs**, **OpenTelemetry**, centralized structured JSON logging, **Prometheus/Grafana** |
| **Operational & Deployment Complexity** | Managing dozens of pipelines, configs, and dynamic IPs | **Docker**, **Kubernetes**, **API Gateway**, **Service Discovery**, **Service Mesh (Istio/Linkerd)** |
| **Testing Difficulty** | Full E2E environments are slow, flaky, and expensive | **Unit + Component tests**, **Testcontainers**, **Consumer-Driven Contract Testing (Pact)**, **Chaos Engineering** |
| **API Evolution & Compatibility** | Independently deployed services can break consumers | **OpenAPI contracts**, **Semantic Versioning**, backward-compatible additive changes, **Expand-and-Contract** |
| **Expanded Security Surface** | Traffic traverses networks instead of in-memory calls | **OAuth2 / OIDC / JWT**, **mTLS**, **Zero-Trust architecture**, centralized secrets management |

---

#### Quick Interview Revision Checklist (Section 1.1)
- Can you explain **Monolith vs Modular Monolith vs SOA vs Microservices** clearly with trade-offs?
- Do you know why **ESB ("smart pipes")** in SOA differs from **"smart endpoints, dumb pipes"** in microservices?
- Can you define a **distributed monolith** and name 3 symptoms that create one (shared DB, lockstep deployments, deep synchronous call chains)?
- Can you connect **fault isolation** to **cascading failure prevention** (timeouts, bulkheads, circuit breakers)?
- Can you recite the core **Distributed Systems realities** (8 Fallacies, CAP/PACELC, partial failures, idempotency)?
- Can you justify **when NOT to use microservices** (startups, small teams, missing DevOps maturity, unclear bounded contexts)?


---

### 1.2 Service Decomposition

---

#### 1. How to Identify Service Boundaries

Defining service boundaries is the **single most important architectural decision** in microservices. Bad boundaries create chatty networks, distributed transactions, and lockstep deployments.

##### Core Boundary Identification Criteria
1. **High Cohesion & Loose Coupling:** Group operations and data that change together for the same business reason inside one service; minimize dependencies across services.
2. **Business Capability & Bounded Context Alignment:** Model boundaries around domain capabilities (`Order Management`, `Payment Processing`, `Inventory`) rather than technical layers (`ControllerService`, `ValidationService`).
3. **Data Ownership & Transactional Boundaries:** Entities that must change atomically together inside a single ACID transaction belong in the **same** service.
4. **Rate of Change (Release Cadence):** Separate volatile features (e.g., `Pricing/Promotions` updated daily) from stable core capabilities (`Ledger/Accounting` updated quarterly).
5. **Independent Scaling & Resource Profile:** Separate CPU/memory-intensive workloads (`VideoTranscoding`, `SearchIndexing`) from standard CRUD APIs.
6. **Security & Compliance Isolation:** Isolate regulated domains (e.g., `PCI-DSS Payment Vault`, `PII Identity Service`) so compliance audits and strict access controls apply only to a small footprint.
7. **Team Autonomy (Conway's Law):** *"Organizations design systems that mirror their communication structures."* Align service boundaries so one small team owns a service end-to-end without waiting on other teams.

> **Anti-Pattern Warning (Entity-per-Table Decomposition):** Never blindly create one microservice per database table (e.g., `OrderService` + `OrderItemService` + `OrderAddressService`). That causes excessive network chatter and distributed transactions over what should be one Aggregate.

---

#### 2. Decomposing a Monolith into Microservices

Never attempt a **"Big Bang" rewrite** (rewriting the whole monolith from scratch in parallel—it almost always stalls or fails). Always decompose **incrementally**.

##### Step-by-Step Decomposition Playbook
1. **Analyze & Modularize First:** Identify domain seams, map static dependencies, and remove circular references inside the monolith (e.g., using ArchUnit / Spring Modulith).
2. **Prioritize Extraction Candidates:** Start with modules that:
   - Sit at the **edge** of the dependency graph (fewest incoming dependencies), or
   - Have high change frequency, scaling bottlenecks, or clear pain points (e.g., `NotificationModule`, `SearchModule`, `BillingModule`).
3. **Apply the Strangler Fig Pattern (Traffic Routing):**
   - Put an **API Gateway / Reverse Proxy** in front of the monolith.
   - Route newly extracted endpoints (`/api/notifications/**`) to the new microservice while legacy routes still hit the monolith.
   - Gradually shrink ("strangle") the monolith piece by piece.
4. **Apply Branch by Abstraction (Internal Code Replacement):**
   - Introduce a clean interface inside the monolith (`InventoryClient`).
   - Keep the local SQL implementation behind it initially, then swap in an HTTP/gRPC client pointing to the new `InventoryService` (using feature flags or shadow/parallel runs to verify parity).
5. **Decompose the Database Last (Hardest Step):**
   - Drop cross-domain foreign-key constraints and replace cross-module SQL `JOIN`s with API calls or replicated read projections.
   - Sync data during transition using **Change Data Capture (CDC / Debezium)** or dual-write/backfill scripts until the new service becomes the sole writer, then drop legacy tables.

```text
Phase 1 (Before)           Phase 2 (Strangler Fig)           Phase 3 (Extracted)
┌────────┐                 ┌────────┐   ┌─────────────┐      ┌────────┐   ┌──────────────┐
│ Client │                 │ Client ├──►│ API Gateway │      │ Client ├──►│ API Gateway  │
└───┬────┘                 └────────┘   └──────┬──────┘      └────────┘   └──────┬───────┘
    ▼                                    ┌─────┴──────┐                   ┌──────┴──────┐
┌──────────────┐                         ▼            ▼                   ▼             ▼
│   Monolith   │                 ┌──────────┐  ┌────────────┐      ┌────────────┐ ┌──────────┐
│ [A] [B] [C]  │                 │ Monolith │  │ Service C  │      │ Service A  │ │Service C │
└──────┬───────┘                 │ [A] [B]  │  └─────┬──────┘      │ Service B  │ └────┬─────┘
       ▼                         └────┬─────┘        ▼             └──────┬─────┘      ▼
 [(Shared DB)]                    [(DB A,B)]     [(DB C)]             [(DB A/B)]   [(DB C)]
```

---

#### 3. Business Capability-Based Decomposition

A **business capability** represents **what a business does** to generate value (its stable core functions), independent of *how* it is currently organized or automated.

- **Key property:** Business capabilities are remarkably **stable** over time—UI layouts, org charts, and workflows change frequently, whereas core capabilities do not.
- **Examples in an E-Commerce System:**
  - `Catalog Management` (managing products, descriptions, categories)
  - `Order Management` (placing, tracking, cancelling orders)
  - `Payment Processing` (authorizing, capturing, refunding payments)
  - `Inventory Management` (stock levels, warehouse allocation)
  - `Customer Management` (profiles, addresses, preferences)
  - `Shipping & Fulfillment` (carrier labels, shipment tracking)

##### Business Capability vs Technical Layer Decomposition (Anti-Pattern)
| Bad: Technical Layer Decomposition | Good: Business Capability Decomposition |
| --- | --- |
| `UI-Service`, `BusinessLogic-Service`, `DatabaseAccess-Service` | `OrderService`, `PaymentService`, `InventoryService` |
| Every feature change requires coordinated edits across all 3 layers/teams | A feature inside Orders is owned and deployed solely by the Order team |

---

#### 4. Domain-Driven Design (DDD)

**Domain-Driven Design (DDD)**, introduced by Eric Evans, is a software design methodology for tackling complex business domains by:
1. Focusing on the **core domain logic** rather than treating architecture as mere database CRUD.
2. Building a **Ubiquitous Language**—a shared, rigorous vocabulary used identically in conversations with domain experts, documentation, class names, methods, APIs, and events.
3. Splitting software design into two complementary levels:
   - **Strategic DDD:** Macro-architecture—finding domain boundaries, bounded contexts, and service-to-service relationships.
   - **Tactical DDD:** Micro-architecture—structuring code inside each service cleanly with entities, value objects, aggregates, domain services, and domain events.

---

#### 5. Strategic DDD

**Strategic DDD** provides tools to map the entire problem space and organize large systems into cleanly separated subsystems and team boundaries. It answers:
- Which parts of the business create competitive advantage vs. commodity plumbing? (**Subdomains**)
- Where does a domain term change its meaning and need a separate model? (**Bounded Contexts**)
- How do autonomous contexts communicate and translate models without polluting each other? (**Context Mapping**)

In microservices architecture, **Strategic DDD is the primary design discipline for finding service boundaries**.

---

#### 6. Subdomains: Core, Supporting and Generic

A business domain is composed of smaller **subdomains**. Strategic DDD classifies them into three types so organizations invest engineering effort wisely:

| Subdomain Type | Definition | Investment & Build Strategy | Example (E-Commerce / Ride-Sharing) |
| --- | --- | --- | --- |
| **Core Subdomain** | The organization's **competitive differentiator** and primary source of business value; complex and constantly evolving. | **Build in-house** with top engineers, strong TDD/DDD tactical modeling, and clean architecture. | Dynamic ride-matching & surge pricing (Uber); personalized recommendations (Netflix); proprietary fraud/risk scoring (FinTech). |
| **Supporting Subdomain** | Custom business logic **necessary for operations**, but **not a market differentiator**. | **Build simply in-house** (standard CRUD microservices/modules); avoid over-engineering. | Vendor onboarding portal, internal product catalog CMS, back-office dispute review tool. |
| **Generic Subdomain** | Well-understood problems **common across almost all businesses** with zero competitive differentiation. | **Buy / Use SaaS or Open-Source** off the shelf (do not reinvent the wheel). | Identity/Auth (`Keycloak`, `Auth0`), Payment Gateway (`Stripe`, `Adyen`), Email/SMS (`SendGrid`, `Twilio`), Logging/Tracing (`OpenTelemetry`). |

---

#### 7. Bounded Contexts

A **Bounded Context** is an explicit semantic boundary within which a particular domain model and its **Ubiquitous Language** apply consistently and unambiguously.

##### Why One Model Fails Across an Enterprise
Trying to force a single "Enterprise `Product` or `Customer` Class" across an entire company causes bloated 100-field entities where different teams conflict over meanings and invariants:
- **`Catalog Context`:** `Product` = title, rich HTML description, images, SEO tags, base list price.
- **`Inventory Context`:** `StockItem` (same SKU!) = warehouse ID, aisle/bin, available quantity, reserved quantity.
- **`Shipping Context`:** `ParcelItem` = weight in grams, dimensions, fragile/hazmat classification.
- **`Order/Billing Context`:** `OrderLineItem` = snapshot price at purchase time, tax rate, discount applied.

Each context models **only what it cares about**, using its own database schema and local vocabulary.

```text
┌─────────────────────────┐    ┌─────────────────────────┐    ┌─────────────────────────┐
│     Catalog Context     │    │    Inventory Context    │    │    Shipping Context     │
│                         │    │                         │    │                         │
│ Product                 │    │ StockItem               │    │ ParcelItem              │
│  • sku                  │    │  • sku                  │    │  • sku                  │
│  • name, description    │    │  • warehouseId          │    │  • weightGrams          │
│  • listPrice, images    │    │  • availableQty         │    │  • dimensions, hazmat   │
└─────────────────────────┘    └─────────────────────────┘    └─────────────────────────┘
```

> **Microservice Boundary Rule:** A single microservice should **never** cross multiple Bounded Contexts. Often **1 Bounded Context = 1 Microservice** (or a cohesive group of closely related microservices owned by one team).

---

#### 8. Context Mapping

**Context Mapping** describes both the organizational and technical integration relationships between Bounded Contexts (Upstream provider vs Downstream consumer).

| Context Mapping Pattern | How It Works | When to Use in Microservices |
| --- | --- | --- |
| **Anti-Corruption Layer (ACL)** | Downstream service writes a translator/adapter layer that converts external upstream DTOs/events into its own clean domain model. | Essential when integrating with legacy monoliths, third-party APIs, or messy upstream services so external models never leak into your core domain. |
| **Open Host Service (OHS) & Published Language** | Upstream service exposes a well-defined, documented, versioned public API/schema (e.g., OpenAPI REST spec, Protobuf, Avro schema registry) for many consumers. | Standard way public/shared platform microservices expose contracts cleanly. |
| **Customer–Supplier** | Upstream (`Supplier`) and Downstream (`Customer`) teams negotiate priorities; upstream tests ensure downstream contracts don't break. | Internal company services with formal APIs and Consumer-Driven Contract Tests (Pact). |
| **Conformist** | Downstream blindly adopts the upstream's data model as-is with no translation layer. | Acceptable only for generic external standards or low-value supporting contexts where translation isn't worth the effort. |
| **Shared Kernel** | Two bounded contexts share a common code module/JAR or schema subset. | Use **very sparingly** in microservices—changes require joint coordination and risk creating a distributed monolith. |
| **Partnership** | Two teams coordinate tightly to build mutually dependent features on the same release cadence. | Temporary alignment during a joint initiative; avoid long-term operational coupling. |
| **Separate Ways** | Contexts have zero integration; small duplicate logic is cheaper than cross-service coupling. | Independent utility features (e.g., duplicating a simple currency formatter rather than calling a service). |

---

#### 9. Tactical DDD

**Tactical DDD** defines the internal building blocks used *inside* a single microservice (Bounded Context) to express rich business logic cleanly and separate domain rules from infrastructure concerns (databases, HTTP frameworks, message brokers—commonly paired with **Hexagonal / Ports & Adapters Architecture**).

Key building blocks: **Entities**, **Value Objects**, **Aggregates & Aggregate Roots**, **Domain Services**, **Domain Events**, and **Repositories**.

---

#### 10. Entities and Value Objects

| Aspect | Entity | Value Object |
| --- | --- | --- |
| **Identity** | Has a **unique, persistent identity (`ID`)** that stays constant across state changes. | **No identity**; defined solely by the combination of its attribute values. |
| **Equality** | **Identity equality:** Two objects with the same `orderId` are the same `Order`, even if status changed. | **Structural equality:** Two `Money(100, "USD")` objects are completely interchangeable. |
| **Mutability** | **Mutable** over its lifecycle (via explicit domain behavior methods, not blind setters). | **Strictly immutable**; changing a value returns a brand-new instance. |
| **Lifecycle** | Tracked over time (`CREATED → PAID → SHIPPED`). | Short-lived; created, passed, and discarded freely. |
| **Java Example** | `Order`, `Customer`, `BankAccount`, `Shipment` | `Money(BigDecimal, Currency)`, `Address`, `OrderId`, `DateRange`, `Email` (`record`) |

```java
// Value Object (immutable Java record with self-validation)
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount == null || amount.signum() < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
    }
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

---

#### 11. Aggregates and Aggregate Roots

- **Aggregate:** A cluster of closely related Entities and Value Objects treated as a **single atomic unit of data consistency** (transactional boundary).
- **Aggregate Root:** The single top-level Entity representing the Aggregate (`Order` is the root for `OrderLineItem` entities and `ShippingAddress` value object).

##### Four Golden Rules of Aggregates (Vaughn Vernon)
1. **Protect Business Invariants Inside the Boundary:** External callers cannot mutate `OrderLineItem` directly—they must call `order.addItem(sku, qty, price)`, allowing `Order` to enforce rules like *"max 50 items per order"* and recalculate `totalAmount` atomically.
2. **Design Small Aggregates:** Avoid gigantic object graphs (e.g., `Customer` containing a list of every `Order` ever placed). Large aggregates cause database lock contention and memory bloat.
3. **Reference Other Aggregates by ID Only:** An `Order` stores `CustomerId customerId` (a Value Object), **never** a direct JPA `@ManyToOne Customer customer` object graph. This keeps boundaries clean and makes extracting microservices trivial later.
4. **One Transaction = One Aggregate Instance:** Modify one Aggregate per local ACID transaction; coordinate changes across multiple Aggregates using **Domain Events** and **eventual consistency** (or Sagas across services).

```text
┌─────────────── Order Aggregate (Consistency Boundary) ───────────────┐
│                                                                      │
│   «Aggregate Root»                                                   │
│   ┌──────────────────────────────────┐                               │
│   │ Order                            │                               │
│   │  • OrderId id                    │───► CustomerId (ID ref only!) │
│   │  • OrderStatus status            │                               │
│   │  • Money totalAmount (invariant) │                               │
│   │  • ShippingAddress (Value Obj)   │                               │
│   └────────────────┬─────────────────┘                               │
│                    │ 1..* (owned internally; no direct external ref) │
│                    ▼                                                 │
│   ┌──────────────────────────────────┐                               │
│   │ OrderLineItem (Child Entity)     │                               │
│   │  • lineId, productId, qty, price │                               │
│   └──────────────────────────────────┘                               │
└──────────────────────────────────────────────────────────────────────┘
```

---

#### 12. Domain Services

A **Domain Service** encapsulates **stateless domain business logic** that operates across multiple Entities/Aggregates or represents a domain concept that does not naturally belong inside any single Entity or Value Object.

- **Examples:** `FundsTransferService` (debiting source `Account` and crediting destination `Account`), `PricingDiscountEngine` (applying customer tier + seasonal promotions + tax rules).

##### Domain Service vs Application Service (Crucial Distinction)
| Domain Service | Application Service (Use-Case Orchestrator) |
| --- | --- |
| Holds **pure business rules/calculations** involving domain objects. | Coordinates a workflow step: `@Transactional`, loads aggregates via Repository, invokes domain methods, persists changes, publishes events. |
| Zero dependency on HTTP, JSON, Kafka, or Spring Web. | Interacts with infrastructure ports (DB repositories, message publishers, security context). |

---

#### 13. Domain Events

A **Domain Event** is an immutable record of a meaningful business fact that **has already occurred** inside a domain. Always named in **past tense**.

- **Examples:** `OrderPlacedEvent`, `PaymentAuthorizedEvent`, `InventoryReservedEvent`, `OrderCancelledEvent`.
- **Typical Event Payload:** `eventId` (UUID for idempotency), `eventType`, `occurredAt` (timestamp), `aggregateId`, `schemaVersion`, and essential domain attributes.

##### Two Scopes of Domain Events
1. **Internal Domain Events (Intra-Service):** Decouples side effects within one service (e.g., Spring's `ApplicationEventPublisher` / `@TransactionalEventListener`).
2. **Integration / External Events (Inter-Service):** Published reliably via the **Transactional Outbox** pattern to a message broker (**Kafka / RabbitMQ**) so other microservices react asynchronously without synchronous coupling.

---

#### 14. Avoiding Tightly Coupled Services

Tight coupling happens when changes in one service force simultaneous changes, failures, or performance degradation in another service.

##### Types of Coupling & How to Avoid Them
| Coupling Type | Root Cause (Anti-Pattern) | How to Decouple |
| --- | --- | --- |
| **Data / Schema Coupling** | Reading/writing another service's database tables directly | Strict **Database per Service**; expose data only via APIs or events. |
| **Temporal (Runtime) Coupling** | Service A calls B → C → D synchronously on the critical path | Use **async messaging (Kafka)** for non-blocking steps; cache read data locally via **Event-Carried State Transfer**. |
| **Contract / Model Coupling** | Returning internal JPA entities directly as REST JSON or sharing domain JARs | Separate **public DTOs / OpenAPI / Protobuf contracts** from internal domain entities; use **Anti-Corruption Layers**. |
| **Chatty / Fine-Grained Coupling** | Service A makes 10 sequential RPC calls to Service B to build one screen | Design **coarse-grained, capability-oriented APIs** (or use BFF/Aggregator/Batch endpoints). |

---

#### 15. Avoiding Distributed Monoliths

A **distributed monolith** is an architecture that splits code into separate deployable processes over a network, **yet all services remain tightly coupled**—delivering all the latency, operational overhead, and failure modes of distributed systems with none of the agility or fault isolation of microservices.

##### 5 Classic Symptoms of a Distributed Monolith
1. **Lockstep Deployments:** Upgrading `OrderService` always requires deploying `InventoryService` and `PaymentService` at the exact same time.
2. **Shared Mutable Database:** Multiple services read/write the same tables and rely on cross-service SQL joins.
3. **Deep Synchronous Call Chains:** `A → B → C → D → E` for every user request; if `E` slows down or goes down, the entire system fails.
4. **Shared Business-Logic Libraries:** A common `core-domain.jar` shared by 10 services—changing one shared model forces recompiling and redeploying all 10 services.
5. **Distributed 2PC / Tightly Orchestrated Locks:** Locking resources synchronously across multiple services while waiting for user/network steps.

##### Prevention Checklist
- Enforce **independent CI/CD**—if two services consistently deploy together, merge them back or fix their contract boundary.
- Enforce **strict database isolation** (separate DB credentials per service).
- Use **Consumer-Driven Contract Testing** (Pact / Spring Cloud Contract) and **backward-compatible additive API versioning**.
- Replace deep synchronous request chains with **asynchronous events, Sagas, and local read projections**.
- Share only **zero-domain-logic technical utilities** (logging formatters, OpenTelemetry wrappers, auth token parsers) or generated API/event schemas.

---

### 1.3 Architecture Decisions

---

#### 1. Database per Service

In microservices, **each service owns its private database (or isolated schema)**, and **no other service is permitted to access that database directly**. All reads and writes go exclusively through the owning service's API or published events.

##### Three Common Deployment Topologies (Logical Isolation vs Cost)
1. **Private Tables / Schema per Service (Same DB Server):** Lowest infrastructure cost and easiest ops; enforce strict separation using **dedicated DB users/roles with permissions restricted to only their own schema** (never share credentials or run cross-schema joins).
2. **Separate Logical Database per Service (Shared Cluster):** Good balance of operational simplicity and logical isolation.
3. **Dedicated Database Server/Engine per Service:** Full **resource & fault isolation** (noisy-neighbor protection) and enables **Polyglot Persistence**:
   - `OrderService` → **PostgreSQL** (relational integrity, ACID)
   - `ProductCatalogService` → **MongoDB / Elasticsearch** (flexible attributes, full-text search)
   - `Session/CartService` → **Redis** (sub-millisecond key-value lookup)
   - `Analytics/IoTService` → **Cassandra / ClickHouse** (high-throughput time-series writes)

##### Benefits
- Internal schema refactoring (renaming columns, splitting tables, adding indexes) never breaks other services.
- Services scale their data tier independently (read replicas, sharding, connection pool tuning).

---

#### 2. Shared Database Anti-Pattern

The **Shared Database Anti-Pattern** occurs when multiple microservices directly read from and/or write to the same database tables.

```text
  [ ANTI-PATTERN: Shared Database ]           [ CORRECT: Database per Service ]
┌───────────┐       ┌───────────┐           ┌───────────┐       ┌───────────┐
│ Order Svc │       │Billing Svc│           │ Order Svc │       │Billing Svc│
└─────┬─────┘       └─────┬─────┘           └─────┬─────┘       └─────┬─────┘
      │  direct SQL       │  direct SQL           │  REST/Kafka       │
      └─────────┬─────────┘                       ▼                   ▼
                ▼                           [(Order DB)]        [(Billing DB)]
       [( Shared Orders DB )]
```

##### Why It Breaks Microservices
1. **Runtime Coupling:** If `OrderService` renames `status` to `order_state`, `BillingService`'s native SQL query crashes in production.
2. **Bypassed Business Invariants & Events:** `BillingService` updates `orders.status = 'PAID'` directly in SQL, bypassing `OrderService`'s domain validation and failing to trigger `OrderPaidEvent`.
3. **Noisy Neighbor & Lock Contention:** A heavy reporting query from `AnalyticsService` locks rows or exhausts DB connections, stalling `CheckoutService`.
4. **Impossible Polyglot Persistence or Independent Scaling:** All teams remain tethered to one database vendor, upgrade schedule, and bottleneck.

> **Exception Rule:** Only during a **temporary monolith migration** (e.g., Strangler Fig transition) might a monolith and new service temporarily share a database while views/CDC decouple them—with a strict deadline to split.

---

#### 3. API Composition

When data is split across databases (`Order DB`, `Customer DB`, `Payment DB`), you can no longer write a SQL `JOIN`. **API Composition** solves cross-service read queries by fetching data from each owning service and joining it **in memory**.

##### How It Works
- An **API Composer** (either an API Gateway, a BFF, a dedicated composition service, or even the client) receives a request like `GET /order-details/{orderId}`.
- It calls `OrderService`, `CustomerService`, and `PaymentService` **in parallel** (e.g., Java `CompletableFuture.allOf(...)`, Virtual Threads, or Spring `WebClient` `Mono.zip(...)`), merges the responses into a single JSON DTO, and returns it.

##### Pros & Cons
- **Pros:** Simple to implement, always returns **fresh real-time data** without duplicating databases or managing eventual consistency sync pipelines.
- **Cons / Limitations:**
  - **Higher latency & lower availability:** Overall latency equals the slowest downstream call (`max(latencies)`), and if one required service fails, the query degrades or fails.
  - **Poor fit for large dataset joins, complex filtering, or pagination across services:** Loading 50,000 orders from Service A and 50,000 customers from Service B to filter/sort in memory does not scale (use **CQRS Materialized Views** instead).

---

#### 4. Backend for Frontend (BFF)

The **Backend for Frontend (BFF)** pattern creates **separate, dedicated backend facade services for each specific client experience** (e.g., `Mobile-BFF`, `Web-BFF`, `Partner-API-BFF`), rather than forcing all clients to use one bloated "one-size-fits-all" API.

```text
┌──────────────┐       ┌────────────┐
│  Mobile App  ├──────►│ Mobile BFF ├──┐
└──────────────┘       └────────────┘  │     ┌───────────────┐
                                       ├────►│ Order Service │
┌──────────────┐       ┌────────────┐  │     ├───────────────┤
│   Web SPA    ├──────►│  Web BFF   ├──┼────►│Catalog Service│
└──────────────┘       └────────────┘  │     ├───────────────┤
                                       └────►│  User Service │
                                             └───────────────┘
```

##### Why Use a BFF?
- **Tailored Payloads:** A mobile screen needs a tiny, bandwidth-friendly JSON payload combining product summary + stock badge + user avatar in **1 network round-trip**, whereas a desktop Web admin dashboard needs full tables, pagination metadata, and extra columns.
- **Independent Release Cycles:** The iOS/Android team can evolve `Mobile-BFF` without breaking Web or forcing core domain teams (`OrderService`) to add UI-specific formatting logic.

##### API Gateway vs BFF
- **API Gateway:** Handles **infrastructure / cross-cutting edge concerns** across all APIs (TLS termination, routing, rate limiting, JWT validation, WAF).
- **BFF:** Handles **client-specific UI orchestration, API composition, and response shaping** (lives *behind* the API Gateway).
- **Golden Rule:** Keep BFFs thin—**never put core business rules or persistent domain databases inside a BFF**.

---

#### 5. Aggregator Pattern

The **Aggregator Pattern** uses a specialized service (or gateway endpoint) to act as a **single facade** that invokes multiple downstream microservices, combines their responses (and optionally applies business transformation or workflow logic), and returns one consolidated response to the caller.

##### Three Common Variations
1. **Scatter–Gather (Parallel):** Dispatches concurrent requests to `FlightService`, `HotelService`, and `CarRentalService`, collects all responses within a timeout budget, and merges them.
2. **Chained:** Calls `OrderService` first to get `customerId` and `paymentId`, then calls `CustomerService` and `PaymentService` using those IDs.
3. **Branching:** Calls `UserService` and, based on `user.accountType`, routes conditionally to `RetailPricingService` or `EnterprisePricingService`.

##### Production Best Practices for Aggregators
- **Parallelize independent calls** (`CompletableFuture`, Java 21 Virtual Threads, or `WebClient` reactive `Mono.zip`).
- **Apply strict timeouts and fallbacks (Graceful Degradation):** If `RecommendationsService` or `LoyaltyPointsService` times out at 150ms, return the core order/product data with `recommendations: []` instead of failing the entire request with HTTP 500.
- **Avoid "God Aggregators":** Don't let an aggregator accumulate heavy domain logic or direct DB state; otherwise it turns back into a monolith.

---

#### 6. Service Autonomy

**Service autonomy** measures how independently a microservice can **develop, deploy, scale, and serve requests at runtime** without depending on the synchronous availability or release schedule of other services.

##### Development-Time vs Runtime Autonomy
- **Development/Deployment Autonomy:** Team owns its repo, CI/CD, database schema, and backward-compatible APIs—no coordinated releases needed.
- **Runtime Autonomy:** Even if `CustomerService` or `InventoryService` goes down temporarily, `OrderService` can still accept orders (or serve read endpoints) rather than failing immediately.

##### Techniques to Maximize Runtime Autonomy
1. **Asynchronous Communication:** Publish `OrderPlacedEvent` to Kafka instead of blocking synchronously on `EmailService`, `LoyaltyService`, and `AnalyticsService`.
2. **Event-Carried State Transfer (ECST):** Replicate lightweight read-only reference data locally (e.g., `OrderService` caches `customerTier` and `shippingAddress` from `CustomerUpdatedEvent`), eliminating runtime HTTP lookups to `CustomerService`.
3. **Resilience Fallbacks & Circuit Breakers:** Serve cached data or queue work for later when a dependency is degraded.

---

#### 7. Data Ownership

**Data ownership** establishes a **Single Source of Truth (SSOT)** for every piece of business data in the distributed system.

##### Core Rules of Data Ownership
1. **Single Writer Principle:** For every domain entity/attribute, **exactly one microservice** has write authority and enforces its business invariants (e.g., `CustomerService` is the sole owner of customer email/address; `PaymentService` is the sole owner of transaction settlement status).
2. **Consumers Hold Read-Only Projections (Never Authoritative):**
   - `OrderService` may store a **point-in-time snapshot** (`shippingAddress`, `unitPriceAtPurchase`—which legally *should* stay immutable for an invoice!) or a **cached read projection** (`customerName`) synced via Kafka events.
   - `OrderService` must **never** accept API edits to customer profile data or publish authoritative `CustomerUpdated` events.
3. **Prevent Ownership Confusion:** If two services appear to mutate the exact same business concept independently, either:
   - The capability was split prematurely (consider merging them), or
   - They represent **two distinct Bounded Context concepts** that happen to share a name (e.g., `Catalog.ProductPrice` vs `Order.PurchasedLinePrice`).

---

#### 8. Distributed Data Management

Because microservices lack a single shared ACID database, managing reads, writes, and consistency across services requires matching the right pattern to each use case:

| Need / Operation | Recommended Pattern | How It Works & Trade-Off |
| --- | --- | --- |
| **Atomic DB Write + Event Publish (Single Service)** | **Transactional Outbox (+ CDC)** | Write domain state and outbox event row in the **same local DB ACID transaction**; a poller or Debezium CDC publishes to Kafka. Eliminates the dual-write bug. |
| **Multi-Service Write Transaction** | **Saga Pattern** (Choreography or Orchestration) | Sequence of local ACID transactions across services; on failure, execute **compensating transactions** (e.g., `RefundPayment`, `ReleaseStock`) to undo prior steps. |
| **Simple Multi-Service Read (by ID)** | **API Composition** | Query owning services concurrently in memory and merge results. Fresh data, zero sync lag, no extra read DB. |
| **Complex Multi-Service Search / Filter / Join / Pagination** | **CQRS Materialized View** | Subscribe to domain events (`OrderCreated`, `CustomerUpdated`, `PaymentCaptured`) and maintain a dedicated denormalized read DB (PostgreSQL / Elasticsearch) optimized for queries. |
| **Avoid Runtime Lookup Latency for Reference Data** | **Event-Carried State Transfer (ECST)** | Producer publishes full changed state in events; consumers store a local read cache, boosting autonomy and latency at the cost of slight eventual consistency. |

---

#### 9. Service Dependency Management

As microservices grow from 5 to 50+, unmanaged dependencies create fragile "spaghetti graphs," cascading outages, and latency spikes.

##### Key Mathematical Reality — Compounding Availability & Tail Latency
If Service A depends synchronously on 4 sequential/required downstream services, each with **99.9%** availability:
- Combined Availability = $0.999^4 \approx 99.6\%$ (downtime jumps **4x**, from ~8.7 hours/year to ~35 hours/year).
- **Tail Latency Amplification:** If Service A calls 10 services in parallel and each has a 1% chance of taking >500ms (P99), the probability that *at least one* call hits P99 is $1 - 0.99^{10} \approx 9.6\%$—turning P99 into **P90**!

##### Best Practices for Managing Dependencies
1. **Enforce a Directed Acyclic Graph (DAG) — Zero Cyclic Dependencies:**
   - Never allow `Service A → Service B → Service A` (creates infinite loops, thread deadlocks, and startup failures).
   - Break cycles by using **asynchronous events** (Service B emits an event Service A subscribes to) or extracting shared capabilities into a separate lower-level service.
2. **Limit Synchronous Call Depth (Max 1–2 Hops):**
   - Prefer `Gateway/BFF → Service` (1 hop) or `Service A → Service B` (2 hops) and move downstream side effects to async events.
3. **Avoid Shared Domain Libraries (Dependency Hell):**
   - Sharing a `common-models.jar` containing business logic couples release cycles (diamond dependency conflicts across Spring/Jackson versions). Share only **wire contracts** (OpenAPI YAML, Protobuf `.proto`, Avro schemas) or minimal technical observability wrappers.
4. **Automate Dependency Visibility & Contract Verification:**
   - Use **OpenTelemetry Service Maps** (Jaeger/Tempo/Grafana) in production to spot unexpected runtime dependencies and deep call chains.
   - Use **Consumer-Driven Contract Testing (Pact / Spring Cloud Contract)** in CI so providers cannot deploy breaking changes silently.
   - Protect every outbound dependency with **Timeouts, Circuit Breakers, Bulkheads, and Fallbacks** (Resilience4j).

---

#### Quick Interview Revision Checklist (Sections 1.2 & 1.3)
- Can you explain **how to decompose a monolith safely** using **Strangler Fig**, **Branch by Abstraction**, and **database decomposition last**?
- Can you distinguish **Core vs Supporting vs Generic Subdomains** and give concrete build-vs-buy examples?
- Can you explain **Bounded Contexts** (same word, different model—e.g., `Product` in Catalog vs Inventory vs Shipping) and **Context Mapping** (especially **Anti-Corruption Layer**)?
- Do you know the rules of **Entities vs Value Objects** and **Aggregates** (one transaction = one aggregate; reference other aggregates by ID only)?
- Can you explain how **API Composition** differs from **CQRS Materialized Views** for cross-service queries, and when each breaks down?
- Can you contrast **API Gateway vs BFF vs Aggregator** and define **Single Source of Truth (Data Ownership)**?
- Can you explain why **cyclic dependencies** and **deep synchronous call chains** degrade availability and tail latency, and how **ECST + async events** maximize **Service Autonomy**?
