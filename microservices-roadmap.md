# Microservices Roadmap

## 1. Microservices Fundamentals

### 1.1 Architecture fundamentals
- What are microservices?
- Monolithic architecture.
- Modular monolith.
- Service-oriented architecture (SOA).
- Microservices versus monolith.
- Microservices versus SOA.
- Advantages and disadvantages.
- Characteristics of microservices.
- Independent development and deployment.
- Independent scalability.
- Fault isolation.
- Distributed systems fundamentals.
- When not to use microservices.
- Microservices architecture challenges.

### 1.2 Service decomposition
- How to identify service boundaries.
- Decomposing a monolith into microservices.
- Business capability-based decomposition.
- Domain-driven design (DDD).
- Strategic DDD.
- Subdomains: core, supporting and generic.
- Bounded contexts.
- Context mapping.
- Tactical DDD.
- Entities and value objects.
- Aggregates and aggregate roots.
- Domain services.
- Domain events.
- Avoiding tightly coupled services.
- Avoiding distributed monoliths.

### 1.3 Architecture decisions
- Database per service.
- Shared database anti-pattern.
- API composition.
- Backend for Frontend (BFF).
- Aggregator pattern.
- Service autonomy.
- Data ownership.
- Distributed data management.
- Service dependency management.

## 2. Inter-Service Communication

> **Very high priority**

### 2.1 Synchronous communication
- REST API communication.
- HTTP fundamentals.
- REST architectural constraints.
- HTTP methods and status codes.
- Request-response lifecycle.
- HTTP headers and content negotiation.
- JSON serialization and deserialization.
- REST API design principles.
- REST versus gRPC.
- REST versus GraphQL.
- OpenFeign.
- RestClient.
- WebClient.
- Synchronous communication limitations.

### 2.2 HTTP and networking internals
- DNS resolution.
- TCP connection establishment.
- TCP three-way handshake.
- TLS handshake.
- HTTP/1.1 versus HTTP/2.
- HTTP keep-alive.
- Connection reuse.
- Connection pooling.
- Persistent connections.
- Socket exhaustion.
- Connection pool exhaustion.
- Connection timeout.
- Read timeout.
- Write timeout.
- Request timeout.
- DNS caching.
- Connection reset.
- Broken pipe.
- Network partitions.
- Network latency.
- Request propagation across services.

### 2.3 Asynchronous communication
- Message brokers.
- Message queues.
- Publish-subscribe architecture.
- Event-driven architecture.
- Commands versus events.
- Event notification.
- Event-carried state transfer.
- Event choreography.
- Asynchronous processing.
- Message acknowledgements.
- Message delivery guarantees.
- Dead-letter queues.
- Message replay.
- Event schema design.
- Event versioning.
- Schema evolution.
- Backward and forward compatibility.

### 2.4 API design
- API contracts.
- API versioning.
- Backward compatibility.
- Request validation.
- Pagination.
- Filtering and sorting.
- Error response design.
- Standardized error handling.
- API documentation.
- OpenAPI/Swagger.
- Consumer-driven contracts.
- API deprecation strategies.

## 3. API Gateway and Service Discovery

### 3.1 API Gateway
- What is an API Gateway?
- Why do microservices need a gateway?
- Gateway routing.
- Route predicates.
- Request filtering.
- Authentication and authorization.
- Rate limiting.
- Throttling.
- Request transformation.
- Response aggregation.
- Load balancing.
- Gateway security.
- Gateway high availability.
- Gateway as a single point of failure.
- API Gateway versus reverse proxy.
- API Gateway versus service mesh.

### 3.2 Service discovery
- What is service discovery?
- Why is service discovery necessary?
- Client-side discovery.
- Server-side discovery.
- Service registry.
- Service registration.
- Service deregistration.
- Health checks and heartbeats.
- Registry consistency.
- Stale service registrations.
- Dynamic service discovery.
- Service discovery using Eureka.
- Consul fundamentals.
- Kubernetes service discovery.
- Load-balancing algorithms:
  - Round robin.
  - Weighted round robin.
  - Least connections.
  - Random selection.

## 4. Resilience and Fault Tolerance

> **Critical for IBM and Red Hat**

### 4.1 Failure fundamentals
- Partial failures.
- Transient failures.
- Permanent failures.
- Cascading failures.
- Single points of failure.
- Failure isolation.
- Fault tolerance.
- Graceful degradation.
- Fail-fast principle.
- Resilient system design.

### 4.2 Timeout management
- Connection timeout.
- Read timeout.
- Request timeout.
- End-to-end timeout.
- Timeout propagation.
- Timeout budgeting.
- Downstream timeout configuration.
- Timeout versus retry.
- Preventing indefinitely blocked requests.

### 4.3 Retry mechanisms
- Retry policies.
- Retryable versus non-retryable exceptions.
- Fixed interval retry.
- Exponential backoff.
- Exponential backoff with jitter.
- Retry limits.
- Retry budgets.
- Retry amplification.
- Retry storms.
- Safe retry of operations.
- Retry and idempotency relationship.

### 4.4 Circuit breaker
- Why circuit breakers are needed.
- Closed state.
- Open state.
- Half-open state.
- State transitions.
- Failure-rate threshold.
- Slow-call-rate threshold.
- Sliding window.
- Count-based sliding window.
- Time-based sliding window.
- Minimum number of calls.
- Wait duration in open state.
- Permitted calls in half-open state.
- Circuit breaker versus retry.
- Circuit breaker implementation using Resilience4j.

### 4.5 Bulkhead
- Bulkhead pattern.
- Thread-pool bulkhead.
- Semaphore bulkhead.
- Thread isolation.
- Resource isolation.
- Queue capacity.
- Pool exhaustion.
- Bulkhead versus circuit breaker.

### 4.6 Other resilience mechanisms
- Rate limiting.
- Token bucket algorithm.
- Leaky bucket algorithm.
- Load shedding.
- Backpressure.
- Fallback mechanisms.
- Fallback limitations.
- Graceful degradation.
- Health checks.
- Retry and circuit breaker interaction.
- Preventing cascading failures.

## 5. Distributed Transactions and Data Consistency

> **Critical**

### 5.1 Distributed transaction fundamentals
- ACID properties.
- BASE properties.
- Local transactions.
- Distributed transactions.
- Transaction boundaries.
- Database per service.
- Shared database problems.
- Strong consistency.
- Eventual consistency.
- Consistency trade-offs.
- CAP theorem.
- Distributed transaction challenges.

### 5.2 Transaction coordination
- Two-phase commit (2PC).
- Prepare phase.
- Commit phase.
- Coordinator failure.
- Blocking behaviour.
- Limitations of 2PC.
- Saga pattern.
- Saga choreography.
- Saga orchestration.
- Compensating transactions.
- Saga failure handling.
- Saga timeout.
- Compensation failure.
- Try-Confirm-Cancel (TCC).

### 5.3 Data consistency patterns
- CQRS.
- Command model.
- Query model.
- Event sourcing.
- Event replay.
- Materialized views.
- API composition.
- Eventual consistency.
- Data synchronization.
- Conflict resolution.
- Distributed locking.

### 5.4 Transactional messaging
- Dual-write problem.
- Database commit and message publishing failure.
- Transactional Outbox pattern.
- Outbox table.
- Outbox polling.
- Change Data Capture (CDC).
- Transaction log tailing.
- Transactional Inbox pattern.
- Duplicate event handling.
- Outbox cleanup.
- Event publishing guarantees.

## 6. Idempotency and Message Reliability

### 6.1 Idempotency
- What is idempotency?
- Idempotent HTTP methods.
- Idempotent POST operations.
- Idempotency keys.
- Request deduplication.
- Idempotency storage.
- Idempotency expiration.
- Concurrent duplicate requests.
- Payment idempotency.
- Idempotent consumers.

### 6.2 Message delivery guarantees
- At-most-once delivery.
- At-least-once delivery.
- Exactly-once semantics.
- Effectively-once processing.
- Duplicate message delivery.
- Message acknowledgement.
- Offset management.
- Consumer retries.
- Dead-letter queues.
- Poison messages.
- Message ordering.
- Event replay.
- Deduplication strategies.

## 7. Security in Microservices

### 7.1 Authentication and authorization
- Authentication versus authorization.
- JWT.
- JWT structure.
- Access tokens.
- Refresh tokens.
- Token expiration.
- Token validation.
- OAuth 2.0.
- OpenID Connect.
- Role-based access control (RBAC).
- Attribute-based access control (ABAC).
- Keycloak fundamentals.

### 7.2 Service-to-service security
- Service identity.
- Service-to-service authentication.
- Mutual TLS (mTLS).
- Certificate validation.
- Token propagation.
- Token exchange.
- Zero-trust architecture.
- Secure internal communication.
- Secret management.
- Credential rotation.
- Least privilege.

### 7.3 API security
- HTTPS.
- CORS.
- CSRF.
- SQL injection.
- Broken authentication.
- Rate limiting.
- API abuse prevention.
- Input validation.
- Security headers.
- Secure error handling.

## 8. Observability and Production Debugging

> **Very high priority**

### 8.1 Logging
- Centralized logging.
- Structured logging.
- JSON logging.
- Log levels.
- Correlation IDs.
- Request IDs.
- Trace IDs.
- Span IDs.
- Log aggregation.
- Log retention.
- Sensitive data masking.

### 8.2 Metrics
- Application metrics.
- Infrastructure metrics.
- Business metrics.
- Counters.
- Gauges.
- Histograms.
- Throughput.
- Error rate.
- Latency.
- P50, P90, P95 and P99.
- Resource utilization.
- Prometheus.
- Grafana.

### 8.3 Distributed tracing
- What is distributed tracing?
- Trace and span.
- Parent-child spans.
- Trace context propagation.
- Cross-service tracing.
- Asynchronous trace propagation.
- OpenTelemetry.
- Sampling.
- Trace visualization.
- Identifying slow services.

### 8.4 Monitoring and reliability
- Health endpoints.
- Liveness probes.
- Readiness probes.
- Startup probes.
- Alerting.
- Service-level indicators (SLI).
- Service-level objectives (SLO).
- Service-level agreements (SLA).
- RED method.
- USE method.
- Root-cause analysis.
- Incident management.

### 8.5 Production troubleshooting
- API latency suddenly increases.
- HTTP 500 error spike.
- Database connection exhaustion.
- High CPU consumption.
- Memory leaks.
- Thread starvation.
- Downstream service unavailable.
- Kafka consumer lag.
- Increased retry rate.
- Circuit breaker continuously opening.
- Request tracing across multiple services.

## 9. Performance and Scalability

### 9.1 Scalability
- Horizontal scaling.
- Vertical scaling.
- Stateless services.
- Stateful services.
- Load balancing.
- Autoscaling.
- Scaling bottlenecks.
- Database scaling.
- Read replicas.
- Database sharding.
- Caching strategies.

### 9.2 Caching
- Local caching.
- Distributed caching.
- Redis.
- Cache-aside.
- Read-through.
- Write-through.
- Write-behind.
- Cache invalidation.
- Cache expiration.
- TTL.
- Cache stampede.
- Cache penetration.
- Cache consistency.

### 9.3 Performance engineering
- Database connection pooling.
- HTTP connection pooling.
- Thread pools.
- Request concurrency.
- CPU-bound versus I/O-bound workloads.
- Memory utilization.
- Garbage collection impact.
- Serialization overhead.
- Network overhead.
- Database indexing.
- Query optimization.
- N+1 query problem.
- Backpressure.
- Load shedding.
- Performance profiling.

### 9.4 Performance testing
- Load testing.
- Stress testing.
- Soak testing.
- Spike testing.
- Throughput measurement.
- Latency measurement.
- Capacity planning.
- Bottleneck identification.

## 10. Testing Microservices

- Unit testing.
- Integration testing.
- Component testing.
- Contract testing.
- Consumer-driven contract testing.
- End-to-end testing.
- API testing.
- Service virtualization.
- Mocking downstream services.
- Testcontainers.
- Database integration testing.
- Message broker integration testing.
- Failure injection.
- Chaos engineering.
- Resilience testing.
- Performance testing.
- Testing eventual consistency.
- Testing Saga compensation.
- Testing idempotency.
- Testing duplicate events.

## 11. Deployment and Cloud-Native Microservices

### 11.1 Containerization
- Docker fundamentals.
- Docker images.
- Containers.
- Multi-stage builds.
- Container networking.
- Environment variables.
- Configuration management.
- Container health checks.
- Resource constraints.
- Container security.

### 11.2 Kubernetes
- Pods.
- Deployments.
- ReplicaSets.
- Services.
- ClusterIP.
- NodePort.
- LoadBalancer.
- Ingress.
- ConfigMaps.
- Secrets.
- Namespaces.
- Horizontal Pod Autoscaler.
- Resource requests and limits.
- Liveness and readiness probes.
- Service discovery.
- Rolling updates.
- Pod termination.
- Graceful shutdown.

### 11.3 Deployment strategies
- Rolling deployment.
- Blue-green deployment.
- Canary deployment.
- Zero-downtime deployment.
- Rollback strategies.
- Database migration compatibility.
- Backward-compatible releases.
- Version compatibility.
- Service lifecycle management.

### 11.4 Twelve-factor applications
- Codebase.
- Dependencies.
- Configuration.
- Backing services.
- Build, release and run.
- Processes.
- Port binding.
- Concurrency.
- Disposability.
- Dev/prod parity.
- Logs.
- Admin processes.

### 11.5 Service mesh
- What is a service mesh?
- Data plane.
- Control plane.
- Sidecar proxy.
- Istio.
- Linkerd.
- Traffic management.
- Service-to-service security.
- Observability.
- Service mesh versus API Gateway.

## 12. Microservices Design Patterns

You should know the problem each pattern solves, its implementation, limitations and alternatives.

| Pattern | What to understand |
| --- | --- |
| API Gateway | Centralized routing and cross-cutting concerns |
| Service Discovery | Dynamic service location |
| Circuit Breaker | Failure isolation |
| Retry | Recovery from transient failures |
| Bulkhead | Resource isolation |
| Saga | Distributed transaction coordination |
| CQRS | Separate read and write models |
| Event Sourcing | Event-based state reconstruction |
| Outbox | Reliable event publication |
| Inbox | Duplicate message prevention |
| Strangler Fig | Gradual monolith migration |
| Sidecar | Supporting functionality alongside services |
| Ambassador | Proxy-based communication |
| BFF | Client-specific backend |
| Aggregator | Combining multiple service responses |
| Cache-Aside | Cache management |
| TCC | Distributed transaction coordination |
| Leader Election | Coordinating distributed workers |
| Anti-Corruption Layer | Protecting domain boundaries |

## 13. Advanced Java Microservices Engineering

This section is particularly relevant to IBM and Red Hat because their roles involve Java-based backend development.

### 13.1 Application internals
- Dependency injection.
- Inversion of control.
- Application lifecycle.
- Request lifecycle.
- Filters and interceptors.
- Exception handling.
- Validation.
- Serialization.
- Configuration management.
- Environment-specific configuration.

### 13.2 Concurrency
- Threads and thread pools.
- ExecutorService.
- CompletableFuture.
- Synchronization.
- Locks.
- Deadlocks.
- Race conditions.
- Thread safety.
- Concurrent collections.
- Thread starvation.
- Context propagation.
- Blocking versus non-blocking execution.

### 13.3 Spring Boot and Quarkus implementation
- REST controllers and resource classes.
- Service and repository layers.
- Dependency injection.
- Configuration properties.
- REST clients.
- Exception handling.
- Validation.
- Health checks.
- Metrics.
- Distributed tracing.
- Resilience4j.
- Reactive programming.
- Blocking versus non-blocking I/O.
- Quarkus build-time optimizations.
- Native image considerations.
- Spring Boot versus Quarkus.
- JVM versus native execution.

## 14. Real-World Microservices Scenarios

> **Must practise for interviews**

These are not just theoretical questions. You should be able to design, implement and troubleshoot each scenario.

| Scenario | Concepts to apply |
| --- | --- |
| Payment succeeds but order creation fails | Saga, compensation |
| Customer sends payment request twice | Idempotency |
| Database commits but event publication fails | Transactional Outbox |
| Kafka consumer processes the same event twice | Deduplication, Inbox |
| Consumer crashes after processing but before acknowledgement | At-least-once delivery |
| Inventory reaches zero during concurrent purchases | Locking, optimistic concurrency |
| One downstream service becomes slow | Timeout, bulkhead, circuit breaker |
| One service becomes unavailable | Graceful degradation |
| Multiple services fail together | Cascading failure prevention |
| API latency suddenly increases | Distributed tracing |
| Database connections are exhausted | Connection pool management |
| Service deployment breaks existing clients | API compatibility |
| Two events arrive out of order | Event ordering and versioning |
| Compensation transaction fails | Saga recovery |
| One service receives excessive traffic | Rate limiting and load shedding |
| A new service needs to replace a monolith | Strangler Fig |
| Service restarts during active requests | Graceful shutdown |
| One consumer cannot process a message | Retry and DLQ |
| A service is scaled to multiple replicas | Statelessness and shared state management |
| A distributed system needs consistent read models | CQRS and eventual consistency |
