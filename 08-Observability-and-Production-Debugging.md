# Microservices Study Notes — Section 8

## 8. Observability and Production Debugging

> **Very high priority**

---

### 8.1 Logging

---

#### 1. Centralized Logging

In a microservices cluster with 40+ services and ephemeral Kubernetes pods that scale and restart constantly, SSHing into nodes or running `kubectl logs` across pods is impossible.
- **Twelve-Factor Rule:** Microservices never manage log files or rotation on local container disks—they write logs strictly to **`stdout` / `stderr`**.
- **Collection Architecture:** A lightweight node DaemonSet agent (**Fluent Bit**, **Vector**, **Grafana Alloy/Promtail**, or **OpenTelemetry Collector**) tails container runtime logs on every host and ships them asynchronously to a central log store (**OpenSearch / Elasticsearch**, **Grafana Loki**, **Splunk**, **IBM Log Analysis**).

---

#### 2. Structured Logging

Traditional unstructured text logs (`"2026-11-23 Order ord-778 failed for user u-42 in 142ms"`) require fragile regex parsing to query.
- **Structured Logging** emits logs as **explicit key-value fields** (`service`, `orderId`, `userId`, `latencyMs`, `traceId`) so you can filter, aggregate, and compute counts instantly (`service="order-service" AND status>=500 | stats count by errorCode`).

---

#### 3. JSON Logging

**JSON Logging** formats every log event as a single-line JSON object written to `stdout`:

```json
{
  "timestamp": "2026-11-23T10:16:00.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "thread": "http-nio-8080-exec-12",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "correlationId": "req-99812",
  "orderId": "ord-778",
  "message": "Payment gateway timeout",
  "exception": "java.net.SocketTimeoutException: Read timed out..."
}
```

- **Why single-line JSON matters in Java:** Multi-line Java stack traces in plain text split into 50 separate log lines in Fluent Bit, breaking search! JSON serializes the entire `exception` stack trace cleanly inside **one atomic log event**.
- **Java Setup:** Spring Boot 3.4+ built-in `logging.structured.format.console=ecs` (or `logstash`), `logstash-logback-encoder`, or Quarkus `quarkus-logging-json`.

---

#### 4. Log Levels

| Level | Production Usage | Examples |
| --- | --- | --- |
| **`ERROR`** | Unexpected system failures requiring investigation / alerts | Unhandled 500s, DB pool exhausted, DLQ routing |
| **`WARN`** | Recoverable anomalies or approaching limits | Retries firing, circuit breaker tripped `OPEN`, fallback used, deprecated API hit |
| **`INFO`** | **Default prod level** — high-level lifecycle & key business milestones | Pod startup/shutdown, `OrderCreated`, `PaymentAuthorized` (never inside tight loops!) |
| **`DEBUG`** | Diagnostic step-by-step details (**disabled in prod by default**) | Enable dynamically at runtime on 1 pod via Spring Boot Actuator `POST /actuator/loggers/com.app.payment` (`{"configuredLevel":"DEBUG"}`) |
| **`TRACE`** | Ultra-verbose wire/SQL payload dumps | Local troubleshooting only |

---

#### 5. Correlation IDs

A **Correlation ID** ties together every log entry and asynchronous background event belonging to **one logical end-to-end user/business workflow**—even across retries and Kafka topics:
- Stored in SLF4J **MDC (Mapped Diagnostic Context)** (`MDC.put("correlationId", id)`) so every log line automatically inherits it.
- Note: With modern **OpenTelemetry**, `traceId` often serves as the primary correlation key, while a business `correlationId` (`checkoutSessionId`) links multiple separate async traces across minutes/hours.

---

#### 6. Request IDs

A **Request ID** (`X-Request-ID`) uniquely identifies **one specific HTTP request** arriving at the Load Balancer or API Gateway:
- Returned to callers in HTTP response headers and RFC 9457 `ProblemDetail` error bodies so customers/QA can quote it in support tickets.
- Note: If a mobile client retries an HTTP call 3 times, each HTTP attempt may get a distinct `Request ID` (or reuse `X-Request-ID` per attempt) while sharing a business `Idempotency-Key` / `Correlation ID`.

---

#### 7. Trace IDs

A **Trace ID** (128-bit / 32 hex chars in W3C Trace Context, e.g., `4bf92f3577b34da6a3ce929d0e0e4736`) uniquely identifies a **complete distributed execution tree** across all microservices, databases, and queues triggered by one root operation.

---

#### 8. Span IDs

A **Span ID** (64-bit / 16 hex chars, e.g., `00f067aa0ba902b7`) identifies **one individual timed step/operation** inside a Trace (e.g., `OrderController.createOrder` or `POST payment-service/charge`).

| Identifier | Scope | Granularity |
| --- | --- | --- |
| **Request ID** | Single external HTTP hop | Gateway / edge tracking & support tickets |
| **Correlation ID** | Full multi-step business workflow (even across hours/async jobs) | End-to-end business log grouping |
| **Trace ID** | Single distributed call tree across all hops | Cross-service trace waterfall + log linking |
| **Span ID** | One operation/RPC/SQL step within a Trace | Exact timing & parent-child nesting |

---

#### 9. Log Aggregation

```text
App Pod (stdout JSON) ─► Fluent Bit / Vector (DaemonSet) ─► [Optional Kafka Buffer] ─► Loki / OpenSearch ─► Grafana / Kibana
```
- **Why Loki vs OpenSearch:** OpenSearch/Elasticsearch indexes every JSON field (powerful full-text search, higher RAM/disk cost); **Grafana Loki** indexes **only small metadata labels** (`service`, `env`, `level`) and compresses raw log chunks in S3/Object Storage (much cheaper at high scale; filter by `traceId` via LogQL).
- **Logs ↔ Traces Correlation:** Including `traceId` in every JSON log line allows Grafana/Kibana to turn `traceId` into a **1-click hyperlink straight into the distributed trace waterfall**!

---

#### 10. Log Retention

Never keep raw verbose logs on expensive hot NVMe SSDs forever. Use **Tiered Retention Policies (ILM)**:
- **Hot Tier (7–14 days):** Fast SSD storage for active incident triage.
- **Warm Tier (30–90 days):** Slower nodes / compressed index for monthly trend reviews.
- **Cold / Archive Tier (1–7 years):** Cheap encrypted object storage (S3 / Glacier) for **audit/compliance (SOC2, PCI-DSS, HIPAA)** (keep separate immutable security audit logs!).

---

#### 11. Sensitive Data Masking

Logs are often readable by broad engineering/SRE teams—**leaking secrets or PII in logs is a major security violation**:
- **Never log:** Passwords, `Authorization: Bearer <JWT>`, API keys, credit card PAN/CVV, SSNs, or full raw HTTP request bodies on auth endpoints.
- **Enforcement Layers:**
  1. **Code Level:** Custom `toString()` overrides (or `@Mask` Jackson serializers) redacting sensitive DTO fields (`"cardLast4": "****1234"`).
  2. **Logger Level:** Logback/Log4j2 `PatternLayout` / masking converter applying regex scrubbing (e.g., `Bearer [A-Za-z0-9\-._~+/]+=*` → `Bearer ***REDACTED***`).
  3. **Collector Level:** Redaction rules in **Fluent Bit / OpenTelemetry Collector** before logs leave the cluster.

---

### 8.2 Metrics

---

#### 1. Application Metrics

Collected directly inside each microservice via **Micrometer** (`spring-boot-starter-actuator` + `micrometer-registry-prometheus` or Quarkus `quarkus-micrometer-registry-prometheus`):
- HTTP request rate, status code counts, latency buckets (`http_server_requests_seconds`)
- JVM Heap/Metaspace usage, GC pause count & duration (`jvm_gc_pause_seconds`)
- Tomcat / Netty thread pool usage & **HikariCP DB connection pool** (`hikaricp_connections_active`, `pending`, `acquire`)
- Resilience4j (`resilience4j_circuitbreaker_state`, `resilience4j_retry_calls_total`) and Kafka consumer lag.

---

#### 2. Infrastructure Metrics

Collected by **cAdvisor / kube-state-metrics / node_exporter**:
- Container CPU utilization AND **Kubernetes CFS CPU throttling** (`container_cpu_cfs_throttled_periods_total`)
- Memory working set vs `memory.limit`, OOMKill events
- Disk I/O latency & saturation, Network RX/TX drops, TCP retransmits, PostgreSQL active locks & buffer cache hit ratio.

---

#### 3. Business Metrics

Domain-specific metrics that technical HTTP metrics completely miss:
- Examples: `orders_created_total`, `payments_authorized_total{provider="stripe"}`, `checkout_revenue_cents_total`, `fraud_rejections_total`.
- **Why they save outages:** If a silent bug disables the "Pay" button on the frontend or rejects all promo codes with `400`, HTTP `5xx` error rate stays `0%`—only a sudden drop in **`orders_created_total`** catches the incident immediately!

---

#### 4. Counters

A **Counter** is a cumulative metric that **only increases** (or resets to `0` on pod restart):
- Examples: `http_requests_total`, `orders_placed_total`, `kafka_messages_failed_total`.
- **PromQL Rule:** Never graph a raw Counter! Always wrap it in **`rate()`** (per-second rate) or **`increase()`** (total increase over a window), which automatically handle pod restart counter resets:
  ```promql
  sum by (service) (rate(http_requests_total[5m]))
  ```

---

#### 5. Gauges

A **Gauge** represents a current **instantaneous value that can go up or down**:
- Examples: `hikaricp_connections_active`, `jvm_memory_used_bytes`, `resilience4j_bulkhead_available_concurrent_calls`, `kafka_consumergroup_lag`.
- **Note:** Gauges reflect only the snapshot at Prometheus scrape time (e.g., every 15s)—micro-spikes between scrapes can be missed unless tracked via Histograms/Counters.

---

#### 6. Histograms

A **Histogram** samples observations (request durations, payload sizes) and counts them into cumulative **preconfigured buckets** (`le="0.01"`, `le="0.05"`, `le="0.1"`, `le="0.5"`, `le="1.0"`, `le="+Inf"`):
- **Why Histograms beat Summaries in Microservices:**
  - Client-side **Summaries** calculate quantiles inside each individual pod—**you CANNOT mathematically average or aggregate percentiles across 10 pods!**
  - **Histograms** expose additive bucket counts (`_bucket`), allowing Prometheus to compute accurate **fleet-wide percentiles across all pods**!
  ```promql
  histogram_quantile(0.99, sum by (le, service) (rate(http_server_requests_seconds_bucket[5m])))
  ```
- **Critical Metric Trap — High Cardinality Labels:** Never attach unbounded values like `userId`, `orderId`, `traceId`, or raw URL paths with IDs (`/orders/12345`) as Prometheus labels! That creates millions of unique time series and OOM-crashes Prometheus! Always parameterize routes (`/orders/{id}`) and keep high-cardinality IDs in **Logs & Traces**.

---

#### 7. Throughput

**Throughput** measures the rate of completed work per unit time (Requests Per Second — **RPS**, Transactions Per Second — **TPS**, or Kafka messages/sec):
```promql
sum(rate(http_server_requests_seconds_count{application="order-service"}[1m]))
```

---

#### 8. Error Rate

Measures failed requests relative to total traffic (absolute error RPS and percentage):
```promql
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m]))
```

---

#### 9. Latency

**Latency** is the duration from receiving a request to completing the response.
- **Why Averages (`avg`) Lie:** Suppose 99 requests take `10ms` and 1 request takes `10,000ms` (10 seconds). The average is `110ms`—looks completely fine, yet 1% of your users waited 10 seconds! Always monitor **percentiles**, never averages alone.

---

#### 10. P50, P90, P95 and P99

| Percentile | Meaning | What It Reveals in Production |
| --- | --- | --- |
| **P50 (Median)** | 50% of requests are faster than this | Typical baseline experience for the average request |
| **P90** | 90% faster, slowest 10% worse | Early degradation across heavy queries or specific endpoints |
| **P95** | 95% faster, slowest 5% worse | Standard operational latency alert threshold |
| **P99 (Tail Latency)** | 99% faster, slowest 1% worse | Exposes JVM GC pauses, DB lock waits, cold pools, and retry spikes—and remember **tail latency amplifies across microservice fan-outs** ($1 - 0.99^{10} \approx 9.6\%$)! |

---

#### 11. Resource Utilization

Monitor resource headroom before saturation triggers non-linear latency spikes:
- **Kubernetes CPU CFS Throttling Trap:** If a Java pod has `limits.cpu: "500m"` and bursts during startup/GC/JSON parsing, the Linux CFS scheduler forcefully pauses JVM threads for the rest of each 100ms quota period—spiking latency by 200ms even though overall cluster CPU is idle! Many teams set `requests.cpu` accurately and avoid overly tight CPU `limits` (or tune carefully with `container_cpu_cfs_throttled_seconds_total`!).
- **Memory Utilization:** Watch **Container Working Set vs Memory Limit** (not just JVM `-Xmx` heap! Off-heap memory: Metaspace, thread stacks, Netty `DirectByteBuffer`, native libraries can push RSS over container limit and trigger **K8s `OOMKill` Exit Code 137**).

---

#### 12. Prometheus

**Prometheus** is the CNCF standard **pull-based Time-Series Database (TSDB)**:
- **Pull Model:** Periodically scrapes HTTP `/actuator/prometheus` (Spring Boot) or `/q/metrics` (Quarkus) every `15s`–`30s` discovered automatically via Kubernetes **`ServiceMonitor` / `PodMonitor`** annotations.
- **Alertmanager:** Evaluates PromQL alert rules on a schedule and deduplicates, groups, silences, and routes alerts to PagerDuty, Slack, or Opsgenie.
- *(For long-term multi-cluster metric storage across months/years, pair Prometheus with **Thanos**, **Cortex**, or **Grafana Mimir**.)*

---

#### 13. Grafana

**Grafana** is the unified observability visualization & correlation dashboarding layer:
- Connects **Prometheus / Mimir** (Metrics), **Loki** (Logs), and **Tempo / Jaeger** (Traces) in one UI.
- **Exemplars (Metric → Trace Bridge):** Prometheus Histograms can store an actual `traceId` sample alongside high-latency buckets—clicking a single outlier dot on a Grafana P99 latency chart jumps directly into the exact distributed trace of that slow request!

---

### 8.3 Distributed Tracing

---

#### 1. What is Distributed Tracing?

**Distributed Tracing** reconstructs the complete end-to-end execution path, causal hierarchy, and timing breakdown of a single request as it traverses API Gateways, microservices, HTTP/gRPC calls, SQL queries, Redis lookups, and Kafka queues.

---

#### 2. Trace and Span

- **Trace:** The full end-to-end journey of a single request across all services, uniquely identified by one **`traceId`**.
- **Span:** A single named, timed operation within a trace containing:
  - `spanId`, `parentSpanId`, `traceId`
  - Operation name (`POST /api/v1/orders`, `SELECT orders`)
  - Start timestamp & duration
  - **Attributes / Tags** (`http.method=POST`, `http.status_code=500`, `db.system=postgresql`)
  - **Span Events & Status** (exception stack trace, `ERROR` status).

---

#### 3. Parent-Child Spans

Spans form a directed tree (DAG) illustrating exact sequential vs parallel execution and pinpointing bottlenecks immediately:

```text
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736   (Total: 420ms)
├─ [Span A] API Gateway: POST /checkout           |████████████████████████████████████████| 420ms
   └─ [Span B] OrderService: createOrder          |  ████████████████████████████████████  | 390ms
      ├─ [Span C] InventoryService: reserveStock  |    ██████                              |  60ms
      ├─ [Span D] PaymentService: chargeCard      |          ████████████████████████      | 250ms  ◄── Bottleneck!
      │  └─ [Span E] Stripe HTTP POST /charges    |            ████████████████████        | 210ms
      └─ [Span F] OrderDB: INSERT orders + outbox |                                  ███   |  30ms
```

---

#### 4. Trace Context Propagation

To connect Span B (`OrderService`) to Span D (`PaymentService`) across a network hop, the client injects the **W3C Trace Context** HTTP headers (standard across OpenTelemetry, Istio, Spring Boot, Quarkus):

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ▲  ▲                                ▲                ▲
        version traceId (128-bit)          parentSpanId (64-bit) flags (01=sampled)
```
- **`tracestate`:** Optional vendor-specific key-value pairs.
- **`baggage`:** Propagates business context across hops (e.g., `tenantId=acme`)—use sparingly so headers don't bloat!

---

#### 5. Cross-Service Tracing

Two standard approaches in Java microservices (zero manual boilerplate):
1. **Zero-Code Instrumentation (OpenTelemetry Java Agent):** Attach `-javaagent:/opt/opentelemetry-javaagent.jar` in the container entrypoint; bytecode instrumentation automatically traces Spring MVC/WebFlux, Quarkus REST, `RestClient`/`WebClient`/Feign, JDBC/HikariCP, Redis, and Kafka.
2. **Framework-Native Instrumentation:**
   - **Spring Boot 3:** `spring-boot-starter-actuator` + `micrometer-tracing-bridge-otel` + `opentelemetry-exporter-otlp`.
   - **Quarkus:** `quarkus-opentelemetry` (native-image compatible build-time instrumentation!).

---

#### 6. Asynchronous Trace Propagation

Traces often break when requests cross thread pools or Kafka queues unless context is explicitly carried:
1. **Intra-JVM Async Threads (`CompletableFuture`, `@Async`, Reactor):** Standard `ThreadLocal` trace context gets lost across worker threads! Use **Micrometer Context Propagation** (`ContextSnapshot` / `Hooks.enableAutomaticContextPropagation()` in WebFlux) or OpenTelemetry `Context.taskWrapping(...)`.
2. **Kafka / RabbitMQ Messaging:** Producer injects `traceparent` into Kafka **Record Headers**; consumer extracts `traceparent` and starts a **child Span** (or a **Span Link** for batch consumers combining 100 different traces into one DB batch).
3. **Transactional Outbox:** Persist the active `traceparent` inside **`outbox_events.headers`** during `@Transactional` DB write so Debezium/Poller restores the exact parent trace when publishing to Kafka!

---

#### 7. OpenTelemetry

**OpenTelemetry (OTel)** is the CNCF industry-standard observability framework that replaced fragmented proprietary agents (and merged OpenTracing + OpenCensus):
- **Unified Signals:** Single standard API/SDK & wire protocol (**OTLP** over gRPC/HTTP) for **Traces, Metrics, and Logs**.
- **OTel Collector Architecture (`Receivers → Processors → Exporters`):**
  - Services export OTLP to a local **OTel Collector** (DaemonSet or Gateway).
  - Collector batches, masks sensitive attributes, runs **Tail-Based Sampling**, and exports to any backend (**Jaeger**, **Grafana Tempo**, **IBM Instana**, **Datadog**, **Dynatrace**) with **zero vendor lock-in**.

---

#### 8. Sampling

At 20,000 RPS, storing 100% of distributed traces costs more than running the application itself!

| Sampling Strategy | How It Works | Pros & Cons |
| --- | --- | --- |
| **Head-Based Sampling** | Decision (`sampled=true/false` in `traceparent`) is made at the **API Gateway / first service** (e.g., `1%` or `5%` of traffic) | Minimal CPU/network overhead; **Flaw:** Might randomly drop the exact `500` error or slow `4s` outlier you need to debug! |
| **Tail-Based Sampling** *(at OTel Collector)* | All spans stream to an OTel Collector that buffers each trace briefly (e.g., 10s) and **decides after completion** | **Keeps 100% of ERROR traces & slow P99 traces**, plus only `1%` of fast `200 OK` traces! Requires trace-ID consistent routing across Collector pods and more RAM. |

---

#### 9. Trace Visualization

Tools like **Jaeger**, **Grafana Tempo**, **Zipkin**, and **IBM Instana** render traces as:
- **Waterfall / Gantt Charts:** Shows exact timestamps, overlaps, synchronous gaps, retries, and SQL timings.
- **Service Dependency Graphs (Service Maps):** Auto-generated topology map showing which services talk to which, RPS, error percentages, and P99 latencies—instantly revealing hidden coupling or unexpected cyclic calls!

---

#### 10. Identifying Slow Services

How to diagnose latency instantly from a Waterfall trace:
1. **One Wide Leaf Span:** `PaymentService → Stripe API` takes 2,100ms out of 2,200ms → downstream external bottleneck.
2. **"Staircase" Sequential Pattern (N+1 Problem):** 50 tiny sequential `SELECT * FROM order_items` or 20 sequential `GET /products/{id}` spans stacked one after another → replace with batch endpoints (`GET /products?ids=...`), JPA `JOIN FETCH`, or parallel calls (`CompletableFuture` / `Mono.zip`).
3. **Unexplained Empty Gap Between Parent Span & Child Span Start:** Often caused by **HTTP connection pool wait**, **DNS/TLS cold handshake**, **Kubernetes CFS CPU throttling**, or a **JVM Stop-the-World GC pause**!

---

### 8.4 Monitoring and Reliability

---

#### 1. Health Endpoints

Every microservice exposes structured health endpoints (Spring Boot Actuator `/actuator/health`, Quarkus SmallRye Health `/q/health`) returning `200 OK` (`UP`) or `503 Service Unavailable` (`DOWN` / `OUT_OF_SERVICE`), split cleanly into **three distinct Kubernetes probes**.

---

#### 2. Liveness Probes

- **Endpoint:** `/actuator/health/liveness` (`/q/health/live`)
- **Question Answered:** *"Is the process alive and un-wedged, or should Kubernetes **kill and restart** this container?"*
- **Golden Rule:** Check **only local internal JVM health** (deadlocks, fatal unrecoverable state). **NEVER check databases, Redis, Kafka, or downstream APIs in Liveness!** If PostgreSQL blips for 20s and Liveness checks PostgreSQL, Kubernetes restarts all 50 pods simultaneously into a `CrashLoopBackOff` disaster!

---

#### 3. Readiness Probes

- **Endpoint:** `/actuator/health/readiness` (`/q/health/ready`)
- **Question Answered:** *"Can this pod currently accept live user traffic via Kubernetes `EndpointSlices`?"*
- **When it returns `DOWN` (`503`):** During cache/warmup initialization, local DB pool initialization failure, or right after receiving `SIGTERM` (graceful shutdown).
- **Critical Trap:** Do **not** fail Readiness when an *optional* downstream microservice fails (or when a Resilience4j Circuit Breaker opens)—otherwise Kubernetes pulls all your pods out of load balancing, cascading one leaf failure into a total platform outage!

---

#### 4. Startup Probes

- **Endpoint:** `/actuator/health/startup` (`/q/health/started`)
- **Question Answered:** *"Has this slow-starting application finished its initial boot sequence yet?"*
- **Why needed:** Disables Liveness & Readiness checks until the Startup probe passes once (`failureThreshold × periodSeconds` budget, e.g., 60–120s for heavy legacy Spring apps), preventing Kubernetes from killing slow startups prematurely without forcing a sloppy high Liveness timeout during steady-state operation!

| Probe Type | Action on Repeated Failure | Should It Check Local DB? | Should It Check Downstream Microservices? |
| --- | --- | --- | --- |
| **Liveness** | **Kills & restarts** container | **NO** | **NO** |
| **Readiness** | **Removes Pod IP** from Service Load Balancer | Optional *(only if critical & pod can't degrade)* | **NO** |
| **Startup** | Kills container if boot budget expires | No | No |

---

#### 5. Alerting

Effective alerting wakes up on-call engineers **only when users are genuinely impacted or about to be**, avoiding **alert fatigue**:
- **Alert on Symptoms, Not Causes:** Page on **"Checkout 5xx Error Rate > 1% for 5 min"** or **"SLO Error Budget burning 14x faster than normal"**, NOT **"Pod CPU > 75%"**!
- **Every Pageable Alert Must Be:**
  1. **Urgent & User-Impacting** (otherwise make it a Slack notification or Jira ticket).
  2. **Actionable** (include a **Runbook URL**, relevant Grafana dashboard link, and severity `SEV1–SEV4`).
  3. **Deduplicated & Grouped** (via Alertmanager so 1 DB outage sends **1 grouped alert**, not 200 separate pod alerts!).

---

#### 6. Service-Level Indicators (SLI)

An **SLI** is a carefully defined quantitative measurement of service reliability from the **user's perspective**:
- **Availability SLI:**
  $$\text{Availability SLI} = \frac{\text{Valid Non-5xx Responses}}{\text{Total Valid Requests}} \times 100\%$$
- **Latency SLI:** Percentage of valid requests served in $\le 300\text{ms}$.
- **Freshness / Correctness SLI (for async/CQRS):** Percentage of Kafka events projected into Read DB within $\le 5\text{s}$.

---

#### 7. Service-Level Objectives (SLO)

An **SLO** is the **internal engineering target** for an SLI over a rolling compliance window (e.g., 28 or 30 days):
- Example: **"99.9% of `POST /checkout` requests return non-5xx within 500ms over a rolling 30-day window."**
- **Error Budget:** $\text{Error Budget} = 100\% - \text{SLO}$.
  - For **99.9%** ("Three Nines") over 30 days ($43,200\text{ min}$): allowed failure budget = **`43.2 minutes`** (or `0.1%` of requests).
  - For **99.99%** ("Four Nines"): **`4.32 minutes`** per 30 days!
- **How SRE teams use Error Budgets:** If budget remains healthy → ship features rapidly; if budget is exhausted → freeze risky non-reliability releases and focus sprints on stability/tech debt!
- **Multi-Window Burn-Rate Alerts:** Page when burning 14.4x of the 30-day budget in 1 hour (catches acute outages fast without noisy spikes).

---

#### 8. Service-Level Agreements (SLA)

An **SLA** is a **legal/commercial contract** with customers defining financial credits, refunds, or penalties if reliability drops below a threshold:
- **Golden Rule:** Always set your **internal SLO strictly tighter than your external SLA** (e.g., SLA = `99.9%`, internal SLO = `99.95%`), giving engineering an early warning buffer before breaching customer contracts.

---

#### 9. RED Method

Created by Tom Wilkie specifically for **request-driven microservices**, tracking 3 metrics on every service/endpoint:
- **R — Rate:** Requests served per second (RPS).
- **E — Errors:** Failed requests per second / error percentage (`5xx`).
- **D — Duration:** Latency distribution histograms (`P50`, `P95`, `P99`).

---

#### 10. USE Method

Created by Brendan Gregg for **infrastructure & bounded hardware/software resources** (CPU, Memory, Disk, Network, HikariCP DB Pools, Tomcat Thread Pools), checking 3 signals:
- **U — Utilization:** Average percentage of resource capacity actively busy (e.g., 85% CPU, 18/20 DB connections active).
- **S — Saturation:** Amount of queued/waiting work that cannot be served immediately (CPU run-queue, `hikaricp_connections_pending > 0`, Linux CPU CFS throttling).
- **E — Errors:** Count of error events (`OOMKill`, network interface packet drops, `HikariPool` timeout exceptions).

*(Google SRE's **Four Golden Signals** combine these: **Latency, Traffic, Errors, Saturation**.)*

---

#### 11. Root-Cause Analysis (RCA)

Structured investigation after resolving an outage to prevent recurrence:
1. Reconstruct an exact **UTC timeline** combining Alert triggers, Git/ArgoCD deployment timestamps, Grafana metrics, OTel traces, and logs.
2. Apply **5 Whys** (look past the immediate trigger—"Pod crashed"—to systemic gaps: *"Why didn't readiness/circuit breaker/test catch it?"*).
3. Produce a **Blameless Postmortem:** Focus strictly on **missing systemic guardrails, automation, and process gaps**—never blame individuals (blaming breeds hiding mistakes!). Track concrete **Preventive Action Items** with owners and due dates.

---

#### 12. Incident Management

Standard production incident response lifecycle:
1. **Detect & Acknowledge:** Alert fires → On-call acknowledges within SLA (e.g., 5 min).
2. **Triage & Declare Severity (`SEV1`–`SEV4`):** Establish **Incident Commander (IC)** (coordinates response, avoids getting bogged down typing commands), **Technical Lead / SMEs** (debug & mitigate), and **Comms Lead** (updates status page & stakeholders).
3. **Mitigate FIRST, Debug Later!** Restore user service ASAP before spending 2 hours hunting root cause:
   - Roll back recent deployment (`kubectl rollout undo` / ArgoCD rollback)
   - Toggle feature flag kill switch
   - Scale pods / Shed load / Force-open broken circuit breaker / Failover DB
4. **Verify Recovery & Monitor Stability.**
5. **Conduct Blameless Postmortem & Follow-Through.**

---

### 8.5 Production Troubleshooting

*(High-frequency senior Java / IBM / Red Hat interview troubleshooting scenarios!)*

---

#### 1. API Latency Suddenly Increases
- **First Checks:**
  1. Check **RED dashboard** & recent **deployments / config / feature-flag changes** in the last 30–60 minutes.
  2. Open **Grafana Tempo / Jaeger** (or click an Exemplar on the P99 spike) to inspect the **Trace Waterfall**—see *which exact span* grew (`App CPU` vs `SQL query` vs `Downstream HTTP call` vs `Gap before span start`).
- **Common Root Causes & Fixes:**
  - **Downstream API / DB slow:** Missing SQL index (query shifted from Index Scan to Seq Scan as table grew!), DB lock contention (`pg_stat_activity`), or N+1 Hibernate lazy queries (`SELECT` repeated 100x in a loop → fix with `JOIN FETCH` / `@EntityGraph`).
  - **JVM / K8s Resource Squeeze:** K8s **CPU CFS Throttling** (`container_cpu_cfs_throttled_seconds_total`) or long **GC Stop-the-World pauses**.
  - **Cache Cold / Eviction Stampede:** Redis restarted or TTL expired simultaneously across popular keys.

---

#### 2. HTTP 500 Error Spike
- **First Checks:**
  1. Correlate timestamp with latest CI/CD release or upstream dependency deploy.
  2. Query centralized **JSON logs** (`service="order-service" AND level="ERROR"`) aggregated by **exception class & stack trace**, and grab a sample `traceId`.
- **Common Root Causes & Fixes:**
  - Bad code release (`NullPointerException`, broken serialization, schema mismatch after DB migration) → **Immediate rollback**!
  - Expired secret/API key/TLS certificate or DB credential rotation mismatch.
  - Unhandled downstream `4xx`/`5xx` propagating out of `RestClient`/`Feign` without a `@RestControllerAdvice` handler or fallback.

---

#### 3. Database Connection Exhaustion
- **Symptoms:** Logs flooded with `SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30000ms`; `hikaricp_connections_active == max` and `hikaricp_connections_pending > 0`.
- **First Checks:**
  1. Inspect **HikariCP metrics** (`active`, `idle`, `pending`, `usage_seconds`).
  2. Query PostgreSQL directly:
     ```sql
     SELECT pid, state, wait_event_type, wait_event, now() - query_start AS duration, left(query, 120)
     FROM pg_stat_activity
     WHERE state != 'idle'
     ORDER BY duration DESC;
     ```
- **Common Root Causes & Fixes:**
  - **`@Transactional` held open across slow external HTTP/Kafka call!** (DB connection sits idle holding locks for 5–30s waiting on Stripe!) → Move remote HTTP calls outside `@Transactional`.
  - **Slow / Locked SQL queries** (`idle in transaction` or row lock pileup) → Fix query/index, set PostgreSQL `statement_timeout` & `idle_in_transaction_session_timeout`.
  - **Pod Scaling Math Bug:** HPA scales from 5 pods to 40 pods (`40 × 20 poolSize = 800 connections`), exceeding PostgreSQL `max_connections = 300`! → Keep per-pod Hikari pool small (`10–15`) and/or put **PgBouncer / RDS Proxy** in front.
  - **Connection Leak:** Raw `DataSource.getConnection()` or JPA `Stream<T>` not closed in `try-with-resources` (enable `leak-detection-threshold: 5000` temporarily to catch exact stack traces!).

---

#### 4. High CPU Consumption
- **First Checks:**
  1. Compare pod CPU usage vs K8s CPU request/limit (check CFS throttling).
  2. Capture a live **30-second CPU Flame Graph** on the hot pod using **`async-profiler`** (`./asprof -d 30 -f cpu.html <pid>`) or **JDK Flight Recorder** (`jcmd <pid> JFR.start duration=30s filename=/tmp/cpu.jfr`)!
- **Common Root Causes & Fixes:**
  - **GC Thrashing:** JVM heap is 98% full, so G1GC/ZGC burns 90% of CPU running continuous GC cycles (confirm via `jvm_gc_pause_seconds`!) → fix memory leak or tune heap `-Xmx`.
  - **Tight Retry / Infinite Loop / Catastrophic Regex Backtracking.**
  - **Heavy JSON Serialization / Compression / TLS Handshakes** (missing HTTP Keep-Alive connection pooling!).

---

#### 5. Memory Leaks
- **Symptoms:** Sawtooth heap graph where post-GC baseline climbs steadily (`60% → 75% → 90% → 99%`) ending in `OutOfMemoryError: Java heap space`, OR container RSS hits K8s `memory.limit` and gets **killed silently by Linux kernel (`OOMKilled`, Exit Code `137`)**!
- **First Checks:**
  1. Check whether the leak is **JVM Heap** (`jvm_memory_used_bytes{area="heap"}`) vs **Native Off-Heap** (Metaspace, DirectByteBuffers, Thread stacks → enable `-XX:NativeMemoryTracking=summary` and run `jcmd <pid> VM.native_memory summary`).
  2. Capture a Heap Dump (`-XX:+HeapDumpOnOutOfMemoryError` or `jcmd <pid> GC.heap_dump /tmp/heap.hprof`) and analyze **Dominator Tree / Leak Suspects** in **Eclipse MAT**.
- **Common Root Causes & Fixes:**
  - Unbounded static `Map`/`List` or in-memory cache without `maximumSize` / TTL eviction (always use Caffeine/Redis with strict max bounds!).
  - Unclosed `ThreadLocal` variables on pooled Tomcat/worker threads (`ThreadLocal.remove()` missing in `finally`).
  - Hibernate 1st-level persistence context accumulating 500k entities inside one massive `@Transactional` batch loop (call `entityManager.flush(); entityManager.clear()` periodically, or chunk).
  - Netty `ByteBuf` / `InputStream` reference-counting leaks.

---

#### 6. Thread Starvation
- **Symptoms:** Pod is alive and **CPU is near 0–5%**, yet API requests hang and time out (including actuator/metrics endpoints if Tomcat worker pool is full!).
- **First Checks:** Take **3 consecutive thread dumps 5 seconds apart** (`jcmd <pid> Thread.print > td1.txt` or `jstack <pid>`) and inspect with **fastThread** / grep thread states:
  - Many threads **`RUNNABLE` at `SocketInputStream.socketRead0`** → **Missing HTTP client Read Timeout** waiting on a hung downstream service!
  - Many threads **`BLOCKED (on object monitor)`** → Heavy `synchronized` lock contention or **deadlock** (`jstack` flags deadlocks automatically at the bottom!).
  - Many threads **`WAITING` / `TIMED_WAITING`** → Waiting on exhausted HikariCP pool, `CompletableFuture.get()`, or Netty event-loop blocked by a accidental JDBC/blocking call!
- **Fixes:** Add explicit HTTP/DB timeouts, apply Resilience4j **Bulkheads + Circuit Breakers**, remove blocking I/O from Netty WebFlux event-loop threads (`Schedulers.boundedElastic()`), or enable Java 21 **Virtual Threads**.

---

#### 7. Downstream Service Unavailable
- **First Checks:**
  1. Check downstream pod status & events (`kubectl get pods`, `kubectl describe pod`, `kubectl get endpointslices`) — look for `CrashLoopBackOff`, `OOMKilled`, failing readiness probes, or broken deploy.
  2. Check DNS resolution, Kubernetes `NetworkPolicy`, expired TLS/mTLS certificates, or Istio routing rules.
- **Expected Healthy Upstream Behavior:** Upstream Circuit Breaker trips `OPEN`, Fallback/Graceful Degradation serves cached/default data (or queues async work via Outbox), and upstream thread pools remain unblocked!

---

#### 8. Kafka Consumer Lag
- **Symptoms:** `kafka_consumergroup_lag` climbs steadily (`500 → 50,000 → 500,000`), delaying emails, CQRS projections, or Saga steps.
- **First Checks:** Distinguish **Stuck Consumer** vs **Slow Consumer**:
  1. Check consumer logs for **rebalance storms** (`Revoking previously assigned partitions`, `CommitFailedException` due to `max.poll.interval.ms` exceeded!) or a **Poison Pill** retrying endlessly on offset `N`.
  2. Check `spring_kafka_listener_seconds` + DB metrics (`HikariCP`, CPU) to see if per-message processing slowed down.
- **Fixes:**
  - **If stuck on poison pill:** Wrap deserializer in `ErrorHandlingDeserializer` + route non-retryables to `.DLT`.
  - **If rebalancing from slow batch:** Lower `max.poll.records` (e.g., `500 → 50`) so batch finishes well inside `max.poll.interval.ms`, or offload long retries to non-blocking `@RetryableTopic`.
  - **If throughput-bound:** Scale consumer pods up to **`<= partitionCount`** (note: adding pods beyond partition count leaves extra pods idle—must increase partitions if already equal!), optimize DB batch inserts (`JDBC batching`), or parallelize processing per key.

---

#### 9. Increased Retry Rate
- **First Checks:** Inspect `resilience4j_retry_calls_total{kind="failed_with_retry"}` alongside downstream latency/5xx metrics.
- **Common Root Causes & Fixes:**
  - Downstream service experiencing partial packet drops, rolling restart, or DB throttling.
  - **Retry Amplification:** Multiple hops (`Gateway + Order + Payment`) retrying simultaneously! → Disable retries on outer hops; retry at one layer only.
  - Misconfigured retry predicate retrying **4xx business errors** (`400`/`404`/`422`) or retrying against an already-`OPEN` Circuit Breaker (`CallNotPermittedException` must be in `ignoreExceptions`!).
  - Missing jitter causing synchronized retry storms → enable Exponential Backoff + Full Jitter + Retry Budgets.

---

#### 10. Circuit Breaker Continuously Opening
- **First Checks:** Inspect `resilience4j_circuitbreaker_state` (`0=CLOSED, 1=OPEN, 2=HALF_OPEN`) and `resilience4j_circuitbreaker_failure_rate` vs `slow_call_rate` in Grafana.
- **Common Root Causes & Fixes:**
  1. **Downstream genuinely degraded:** Let breaker stay `OPEN`; verify downstream health & capacity before touching breaker config.
  2. **Counting 4xx Client Errors as Breaker Failures:** Users sending bad input (`400`/`404`/`422`) falsely trip the breaker! → Exclude business exceptions via `ignoreExceptions`.
  3. **`slowCallDurationThreshold` Too Tight:** Set to `200ms` when downstream normal P95 is `300ms` → calibrate against real Prometheus P99 latency histograms.
  4. **Flapping `OPEN ↔ HALF_OPEN`:** `waitDurationInOpenState` too short (`5s`) or `permittedNumberOfCallsInHalfOpenState` too large (`50`), slamming the recovering downstream pod before its caches/pools warm up.

---

#### 11. Request Tracing Across Multiple Services
End-to-end live production debugging workflow using all 3 observability pillars together:
1. **Grab the `traceId`:** Copy `traceId` from the user's RFC 9457 error response, API Gateway access log, or Grafana P99 Exemplar dot.
2. **Inspect the Distributed Trace (Jaeger / Grafana Tempo):**
   - Open `traceId` waterfall → visually identify **which microservice and span** failed (`status=ERROR`) or consumed 90% of total latency (e.g., `InventoryService` → `HikariPool.getConnection()` took `2,800ms`).
3. **Pivot to Centralized Logs (Loki / OpenSearch):**
   - Filter logs by `traceId="4bf92f3577b34da6a3ce929d0e0e4736"` across **all services** (`Gateway`, `OrderService`, `InventoryService`, and async Kafka consumers!) in chronological order—reading the exact exception message and business IDs (`orderId`, `sku`).
4. **Confirm Scope in Metrics (Prometheus / Grafana):**
   - Check RED & USE dashboards for `InventoryService` around that exact timestamp to verify whether it was an isolated glitch or a cluster-wide DB pool/CPU/deployment issue—then apply targeted mitigation!

---

#### Quick Interview Revision Checklist (Section 8)
- Can you explain **Structured JSON Logging** and distinguish **Request ID vs Correlation ID vs Trace ID vs Span ID**?
- Can you explain why **Prometheus Histograms** aggregate cleanly across pods (via `histogram_quantile` + `rate()`) while **Summaries** do not, and why **high-cardinality labels** crash Prometheus?
- Can you trace a **W3C `traceparent` header** across **HTTP → Outbox Table → Debezium → Kafka → Consumer**, and compare **Head-Based vs Tail-Based Sampling**?
- Can you state the exact rules separating **Liveness vs Readiness vs Startup probes** (and why Liveness must NEVER check PostgreSQL or downstream APIs)?
- Can you define **SLI, SLO, Error Budget, and SLA**, and contrast the **RED Method** (services) vs **USE Method** (resources)?
- For any of the **11 production debugging scenarios** (Latency spike, 500 spike, HikariCP exhaustion, High CPU via `async-profiler`/JFR, Heap/Off-heap Memory leak via MAT/NMT, Thread starvation via `jstack`, Kafka lag, Retry storms, Circuit Breaker flapping), can you name **which metric/tool to check first**, the **root causes**, and the **fix**?
