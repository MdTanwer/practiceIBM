# Microservices Study Notes — Section 11

## 11. Deployment and Cloud-Native Microservices

---

### 11.1 Containerization

---

#### 1. Docker Fundamentals

**Containers** provide lightweight, OS-level process virtualization by sharing the host Linux kernel while isolating each microservice's execution environment:
- **Linux Namespaces (Isolation):** Isolate what a process can *see* (`PID` process tree, `NET` network interfaces/IPs/ports, `MNT` filesystem mounts, `UTS` hostname, `USER` UID mapping).
- **Linux Control Groups (`cgroups`) (Resource Limits):** Restrict how much CPU, memory, and disk/network I/O a container can *consume*.
- **OverlayFS / UnionFS (Copy-on-Write Layers):** Stacks immutable read-only image layers beneath a thin writable container layer.
- **OCI & Podman (Red Hat / OpenShift Ecosystem):** Containers adhere to Open Container Initiative (**OCI**) standards; **Podman** runs daemonless and **rootless** containers seamlessly across RHEL and Red Hat OpenShift without a privileged Docker daemon.

---

#### 2. Docker Images

A **Docker Image** is an immutable, read-only, layered filesystem template containing your microservice binary, JRE runtime, and dependencies:
- **Layer Caching Rule:** Order `Dockerfile` instructions from **least-frequently changed** (base JRE, Maven `pom.xml` dependencies) to **most-frequently changed** (compiled Java application classes)—so code edits reuse cached dependency layers!
- **Spring Boot Layered JARs / Quarkus `fast-jar`:** Split dependencies (`BOOT-INF/lib`) from app classes (`BOOT-INF/classes`) into separate layers for fast CI builds and tiny K8s pulls.
- **Immutability Rule:** Always tag production images with immutable Git commit SHAs or digests (`order-svc:1.4.2@sha256:abc123...`). **Never deploy `:latest` in production!**

---

#### 3. Containers

A **Container** is a live, ephemeral, isolated OS process instantiated from an image with a thin writable scratch layer on top.
- **CRITICAL Java Trap — PID 1 Signal Handling:**
  - **Wrong (Shell Form):** `ENTRYPOINT java -jar app.jar` → Docker spawns `/bin/sh -c "java -jar app.jar"` as **PID 1**. `/bin/sh` swallows `SIGTERM` from Kubernetes, causing the pod to hang for 30s and get brutally `SIGKILL`ed without graceful shutdown!
  - **Right (Exec JSON Form):** `ENTRYPOINT ["java", "-jar", "app.jar"]` → `java` runs directly as **PID 1** and receives `SIGTERM` immediately!

---

#### 4. Multi-Stage Builds

Never ship Maven, JDK compilers, `curl`, `bash`, or `.m2` caches into production! **Multi-stage builds** compile in Stage 1 and copy *only* the runnable JAR/layers into a tiny runtime base image (e.g., **Distroless**, **Alpine JRE**, or **Red Hat UBI 9 Minimal**):

```dockerfile
# Stage 1: Build with full JDK + Maven (~600MB — discarded after build!)
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# Stage 2: Minimal Hardened Production Runtime (~120MB)
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/target/order-service.jar app.jar
USER 1001
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=70.0", "-jar", "/app/app.jar"]
```

---

#### 5. Container Networking

- **Bridge Network:** Default isolated virtual switch inside a single host. On **user-defined bridge networks** (`docker network create app-net`), Docker's embedded DNS resolves containers automatically by container/service name (`jdbc:postgresql://postgres:5432/orders`).
- **Host Network (`--network host`):** Shares host network namespace directly (zero NAT overhead, skips port mapping, loses port isolation).
- **None Network (`--network none`):** Disables all networking (isolated batch/security workloads).
- **Overlay Network:** Multi-host virtual networking across nodes (used under the hood by orchestrators/CNIs).

---

#### 6. Environment Variables

Inject environment-specific values (`SPRING_DATASOURCE_URL`, `KAFKA_BOOTSTRAP_SERVERS`, `JAVA_TOOL_OPTIONS`) at container startup rather than baking environment configs inside the JAR/image (**"Build once, promote exact same image `dev → staging → prod`"**).
- *Security Note:* Avoid passing ultra-sensitive private keys/passwords as plain env vars if avoidable (env vars can appear in crash dumps, `/proc/1/environ`, and `kubectl describe`); prefer **tmpfs-mounted secret files**!

---

#### 7. Configuration Management

Keep immutable application binaries strictly separated from externalized configuration:
- Local defaults live in `application.yml`; environment overrides come from **Kubernetes ConfigMaps / Secrets**, **Spring Cloud Config**, **Vault**, or **GitOps Helm/Kustomize values**.
- Enable live/dynamic config refresh only carefully (`@RefreshScope` or Spring Boot `configtree:` Kubernetes volume reloads).

---

#### 8. Container Health Checks

Docker supports built-in `HEALTHCHECK` instructions in Dockerfiles (useful in Docker Compose / standalone Docker):
```dockerfile
HEALTHCHECK --interval=15s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health/readiness || exit 1
```
*(In **Distroless** images without `wget`/`curl`, use Kubernetes native `httpGet` readiness/liveness probes instead—K8s ignores Dockerfile `HEALTHCHECK` directives anyway!)*

---

#### 9. Resource Constraints

Always bound container CPU and memory (`docker run --memory=1g --cpus=1.5`) so one memory-leaking microservice cannot crash all neighboring containers on the host:
- Modern JVMs (**JDK 11+, 17, 21**) enable **`-XX:+UseContainerSupport`** by default—automatically reading Linux `cgroup` v1/v2 container limits instead of the host VM's 64GB RAM!
- Set **`-XX:MaxRAMPercentage=70.0`** so the heap stays safely inside the container limit alongside off-heap memory (Metaspace, thread stacks, DirectByteBuffers).

---

#### 10. Container Security

Production container hardening checklist (mandatory for enterprise & **Red Hat OpenShift**):
1. **Non-Root User:** Always specify non-root `USER 1001` (OpenShift enforces random arbitrary non-root UIDs automatically; write temp files only to `/tmp`!).
2. **Minimal Base Images:** Use **Distroless**, **Red Hat UBI Micro/Minimal**, or **Alpine**—no `bash`, `curl`, `ssh`, or `sudo` inside production containers!
3. **Read-Only Root Filesystem:** Run containers with `readOnlyRootFilesystem: true` (mount `/tmp` as `emptyDir` if needed).
4. **Vulnerability Scanning & Signing:** Scan images in CI (`Trivy`, `Grype`, `Clair`) to block Critical CVEs, generate **SBOMs**, and sign images with **Cosign / Sigstore** verified by Kubernetes admission controllers (Kyverno/OPA).
5. **Drop Linux Capabilities:** Drop `ALL` kernel capabilities (`CAP_NET_RAW`, `CAP_SYS_ADMIN`).

---

### 11.2 Kubernetes

---

#### 1. Pods

A **Pod** is the smallest deployable compute unit in Kubernetes—wrapping 1 primary microservice container (plus optional **sidecar containers** like an Envoy proxy, Fluent Bit shipper, or Vault Agent) sharing:
- **One shared Pod IP and `localhost` network namespace** (app talks to Envoy sidecar over `127.0.0.1`).
- **Shared ephemeral/mounted Volumes.**
- **Ephemeral by design:** Never create bare `Pod` objects directly in production—always manage them via `Deployment`, `StatefulSet`, or `Job` controllers.

---

#### 2. Deployments

A **Deployment** declaratively manages stateless microservice Pods: desired replica count, image version, **rolling update** strategy (`maxSurge`, `maxUnavailable`), rollout pause/resume, and instant revision rollback (`kubectl rollout undo deployment/order-service`).

---

#### 3. ReplicaSets

A **ReplicaSet** ensures a specified number of identical Pod replicas matching a label selector are alive at any moment—automatically creating a replacement Pod if a node fails or a Pod crashes. `Deployments` create and scale `ReplicaSets` during each release (you interact with `Deployment`, not `ReplicaSet`).

---

#### 4. Services

Because Pod IPs change constantly, a Kubernetes **`Service`** provides a **stable virtual IP (`ClusterIP`), DNS name, and label-selector load balancer** backed dynamically by healthy Pods listed in **`EndpointSlices`** (populated only after Pods pass their `readinessProbe`).

---

#### 5. ClusterIP

The default **`Service`** type (`type: ClusterIP`) exposes a virtual IP reachable **strictly inside the cluster** (`http://order-service.prod.svc.cluster.local:8080`). Ideal for internal East–West microservice traffic behind an Ingress or API Gateway.

---

#### 6. NodePort

Exposes a `Service` on a static high port (`30000–32767`) across **every worker node's IP** (`<NodeIP>:<NodePort>`). Mostly used for local dev testing or underlying Cloud `LoadBalancer` targets—never expose raw `NodePort`s directly to internet users in production.

---

#### 7. LoadBalancer

Provisions a dedicated external Cloud Load Balancer (**AWS NLB/ALB, GCP GLB, Azure LB**) forwarding public internet traffic into cluster Pods. Because creating 50 external Cloud LBs for 50 microservices is expensive, expose only your **API Gateway / Ingress Controller** as `LoadBalancer` (or use Ingress).

---

#### 8. Ingress

An **Ingress** (and its modern successor, the **Kubernetes Gateway API**) provides **Layer-7 HTTP/HTTPS external routing** through a single shared Load Balancer:
- Terminates TLS certificates (`cert-manager`), routes by Host & Path (`/api/v1/orders → order-service:8080`, `/api/v1/payments → payment-service:8080`), and delegates to an **Ingress Controller** (NGINX, Envoy Gateway, HAProxy, Traefik, or **Red Hat OpenShift Routes**).

---

#### 9. ConfigMaps

Stores non-sensitive configuration key-value pairs or entire `application.yaml` / `logback.xml` files separately from container images:
- **Gotcha:** Env vars injected from a `ConfigMap` (`envFrom`) **do NOT update automatically** when you edit the `ConfigMap` (requires rolling restart via Reloader/checksum annotation); ConfigMaps mounted as **volumes** *do* refresh files after ~60s!

---

#### 10. Secrets

Stores sensitive credentials (DB passwords, TLS certificates, OAuth client secrets) injected into pods as tmpfs volume mounts or env vars.
- **Security Reality Check:** Kubernetes `Secrets` are stored in `etcd` merely **Base64-encoded** by default! Always enable **KMS Encryption-at-Rest in `etcd`**, restrict RBAC (`get/list secrets`), and in enterprise production integrate **HashiCorp Vault / Cloud Secret Manager** via **External Secrets Operator (ESO)** or CSI Driver.

---

#### 11. Namespaces

Logical partitions within a shared Kubernetes cluster (`dev`, `staging`, `prod`, `observability`, `ingress-nginx`):
- Scopes resource names & DNS (`svc.namespace.svc.cluster.local`), RBAC permissions, **`ResourceQuota`** (total namespace CPU/RAM caps), **`LimitRange`** (default pod limits), and **`NetworkPolicy`** boundaries.

---

#### 12. Horizontal Pod Autoscaler (HPA)

Automatically scales `Deployment` replicas (`minReplicas` to `maxReplicas`) using observed metrics:
- Standard metrics: CPU / Memory utilization (e.g., target 70% CPU).
- Custom / External metrics (**Prometheus Adapter** or **KEDA**): scale on **HTTP RPS**, **P95 latency**, or **Kafka Consumer Lag**.
- Configure `scaleDown.stabilizationWindowSeconds: 300` to avoid thrashing pods up and down during spiky traffic!

---

#### 13. Resource Requests and Limits

| Setting | Purpose | What Happens If Exceeded? |
| --- | --- | --- |
| **`resources.requests`** | **Guaranteed minimum** reserved by the K8s Scheduler to pick a Node | Pod stays `Pending` if cluster lacks available capacity |
| **`resources.limits.cpu`** | Hard Linux cgroup CPU quota per 100ms period | **CFS CPU Throttling** — JVM threads freeze temporarily, spiking latency! |
| **`resources.limits.memory`** | Hard memory ceiling | Kernel immediately kills pod with **`OOMKilled` (Exit Code 137)**! |

- **QoS Classes:** **`Guaranteed`** (`requests == limits` for CPU & memory — **recommended for critical Java services** so kubelet won't evict them easily!), **`Burstable`** (`requests < limits`), **`BestEffort`** (no requests/limits — evicted first!).

---

#### 14. Liveness and Readiness Probes

```yaml
startupProbe:
  httpGet: { path: /actuator/health/startup, port: 8080 }
  failureThreshold: 30
  periodSeconds: 5       # Up to 150s allowed for slow JVM startup!
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3    # Restarts container ONLY if local JVM is wedged
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8080 }
  periodSeconds: 5
  failureThreshold: 2    # Pulls Pod IP out of EndpointSlice within 10s
```

---

#### 15. Service Discovery (in Kubernetes)

Handled natively via **`CoreDNS` + `Service` (`ClusterIP`) + `EndpointSlices` + `kube-proxy` (iptables/IPVS/Cilium eBPF)**:
- Same-namespace call: `http://payment-service:8080`
- Cross-namespace call: `http://payment-service.payments-prod.svc.cluster.local:8080`
- Use **Headless Service (`clusterIP: None`)** so CoreDNS returns individual Pod IPs directly for **gRPC client-side load balancing** and **Kafka/DB StatefulSets**.

---

#### 16. Rolling Updates

Gradually replaces `v1` pods with `v2` pods without downtime using two key knobs:
- **`maxSurge`** (e.g., `1` or `25%`): How many *extra* pods above `replicas` may run temporarily during rollout.
- **`maxUnavailable`** (set to **`0`** for zero-downtime!): Guarantees K8s spins up and passes `readinessProbe` on a new `v2` pod **before** terminating any healthy `v1` pod!

---

#### 17. Pod Termination

**The Asynchronous Kubernetes Termination Race Condition (Classic Senior Interview Question!):**

When K8s deletes/terminates a Pod at **$t = 0\text{s}$**, **two independent actions happen in parallel**:
1. Control Plane removes the Pod IP from `EndpointSlices` → asynchronously updates `kube-proxy` `iptables`/IPVS and Ingress/Envoy routes across all cluster nodes (**takes ~1–5 seconds**!).
2. **`kubelet` sends `SIGTERM` to PID 1 inside the Pod IMMEDIATELY at $t = 0\text{s}$!**

**What goes wrong without a `preStop` hook?**
If Spring Boot immediately closes its HTTP server at $t = 0.05\text{s}$ upon receiving `SIGTERM`, any external traffic still routed to that Pod IP during the 1–5s `kube-proxy` propagation window hits a closed socket → **`502 Bad Gateway` / `Connection Refused` errors on every deployment!**

---

#### 18. Graceful Shutdown

```text
t = 0s   Pod marked Terminating:
           ├─ (Async) EndpointSlice removed ──► kube-proxy / Envoy updates (~1-5s)
           └─ kubelet runs preStop: sleep 10 ──► Pod KEEPS accepting in-flight routed packets!
t = 10s  kubelet sends SIGTERM to Java PID 1:
           └─ Spring server.shutdown=graceful stops NEW requests, drains active requests, commits Kafka offsets, closes HikariCP!
t = 15s  JVM exits cleanly (Exit 0) well before terminationGracePeriodSeconds (45s) SIGKILL!
```

##### Production Zero-Downtime Deployment Template
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 45   # MUST exceed preStop sleep + Spring shutdown timeout!
      containers:
        - name: order-service
          image: ghcr.io/org/order-service:1.4.2
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]
          env:
            - name: SERVER_SHUTDOWN
              value: "graceful"
            - name: SPRING_LIFECYCLE_TIMEOUT_PER_SHUTDOWN_PHASE
              value: "30s"
```
- Also attach a **`PodDisruptionBudget` (`PDB`)** (`minAvailable: 2` or `maxUnavailable: 1`) so voluntary node drains/upgrades never evict all pods simultaneously!

---

### 11.3 Deployment Strategies

---

#### 1. Rolling Deployment

Incrementally replaces `v1` Pods with `v2` Pods (`maxUnavailable: 0, maxSurge: 1`):
- **Pros:** Built into Kubernetes `Deployment`; needs almost zero duplicate compute infrastructure.
- **Cons:** Both **`v1` and `v2` serve live traffic simultaneously** during rollout (so DB schemas, APIs, Kafka events, and Redis objects **must** be backward/forward compatible!); rolling back takes minutes as new `v1` pods boot back up.

---

#### 2. Blue-Green Deployment

Maintains two complete, identical production environments: **Blue (active `v1`)** and **Green (idle `v2`)**:
1. Deploy `v2` into **Green** and run smoke/sanity tests against an internal preview URL while users stay on **Blue**.
2. Switch the Load Balancer / Service selector / Ingress atomically from `Blue → Green`.
- **Pros:** **Instant 1-second rollback** (flip router back to Blue!), zero mixed `v1 + v2` traffic simultaneously.
- **Cons:** Requires **2x peak compute resources** during release; shared databases still require backward-compatible migrations!

---

#### 3. Canary Deployment

Exposes `v2` gradually to a small percentage of live production traffic before full rollout (`1% → 5% → 25% → 100%`) using **Istio**, **Gateway API**, or **Argo Rollouts**:
- Automated canary analysis compares `v2` vs `v1` **RED metrics** (error rate, P99 latency); if regressions appear, traffic shifts back to `v1` automatically!
- Also supports **Dark Launching** (routing only internal QA users via header `X-Beta-User: true`) or **Traffic Mirroring** (shadowing live prod reads to `v2` silently).

| Strategy | Extra Infra Cost | Blast Radius on Bad Deploy | Rollback Speed | Mixed `v1+v2` Traffic? |
| --- | --- | --- | --- | --- |
| **Rolling** | Low (`+1` or `25%` surge) | Medium–High (replaces fleet quickly) | Moderate (re-spin `v1`) | **Yes** |
| **Blue–Green** | **High (2x fleet)** | High if flipped at 100% (mitigated via smoke tests) | **Instant** (`Green → Blue`) | **No** *(except during DB overlap)* |
| **Canary** | Low–Medium | **Minimal (`1%–5%` users)** | **Instant** (`weight: 0%`) | **Yes** |

---

#### 4. Zero-Downtime Deployment

Achieving zero failed HTTP `5xx`/`502` responses or dropped Kafka messages across deployments requires **5 coordinated layers**:
1. `RollingUpdate` with **`maxUnavailable: 0`** + **`PodDisruptionBudget`**.
2. Strict **`readinessProbe`** (wait until HikariCP/caches/JIT warmup finish).
3. **`preStop` sleep (`10s`) + Spring/Quarkus graceful shutdown (`30s`) + `terminationGracePeriodSeconds: 45`**.
4. **Backward-compatible APIs & Kafka event schemas**.
5. **Non-blocking, Expand-and-Contract DB migrations**.

---

#### 5. Rollback Strategies

- **Kubernetes / GitOps Rollback:** `kubectl rollout undo`, ArgoCD/Flux Git revert, Blue-Green traffic flip, or Canary auto-abort.
- **Feature Flags (LaunchDarkly / Unleash / OpenFeature):** Decouples **Code Deployment** from **Feature Release**—flip a runtime toggle to disable a buggy feature in `< 1 second` without redeploying any container!
- **Critical DB Warning:** You **cannot safely run destructive "down" migrations** (`DROP COLUMN`, `DROP TABLE`) during an emergency rollback without losing newly written production data—always keep DB changes backward-compatible so `v1` code still runs cleanly against the upgraded schema!

---

#### 6. Database Migration Compatibility

Never run breaking DDL in one step (like `ALTER TABLE orders RENAME COLUMN name TO full_name` or `ADD COLUMN tax DECIMAL NOT NULL` without a default) while `v1` pods are live! Use the **Expand-and-Contract (Parallel Change) Pattern** via Flyway / Liquibase:

##### 3-Phase Expand-and-Contract Example (Splitting `full_name` → `first_name` + `last_name`)
1. **Phase 1 — Expand (Backward-Compatible DDL):**
   - Migration V2: `ALTER TABLE customers ADD COLUMN first_name VARCHAR(100), ADD COLUMN last_name VARCHAR(100);` *(nullable! `v1` still works!)*.
   - Deploy App `v2`: **dual-writes** both `full_name` AND (`first_name`, `last_name`), while still reading safely with fallback.
   - Run a batched background backfill job for historical rows.
2. **Phase 2 — Switch Reads:**
   - Once 100% backfilled and `v1` pods are gone, deploy App `v2.1` reading exclusively from `first_name`/`last_name` (while still writing `full_name` temporarily in case you need to roll back!).
3. **Phase 3 — Contract (Cleanup in Future Release):**
   - Days later, after rollback risk has passed, deploy `v3` dropping writes to `full_name`, then run Migration V3 (`DROP COLUMN full_name`).
- **Locking Tip (PostgreSQL):** Always use `CREATE INDEX CONCURRENTLY` outside Flyway's single-transaction wrapper (`-- Flyway: executeInTransaction=false`) so building an index on a 50M-row table doesn't lock out live writes!

---

#### 7. Backward-Compatible Releases

Every deployment must tolerate talking to N-1 versions during rolling/canary overlaps:
- **REST/JSON:** Only additive optional fields; `@JsonIgnoreProperties(ignoreUnknown = true)`.
- **Kafka Events:** Full-compatible Avro/Protobuf/JSON Schema evolution (optional fields with defaults).
- **HTTP Caching / Redis Serialization:** Avoid Java native serialization (`Serializable`) in Redis! Serialize explicit versioned JSON/Protobuf DTOs and **version cache keys** (`v2:order:123` instead of `order:123`) so `v2` adding a non-nullable field doesn't crash `v1` deserializing from the same Redis key!

---

#### 8. Version Compatibility

Managing compatibility matrices across polyglot services and shared dependencies:
- Enforce **Semantic Versioning (`MAJOR.MINOR.PATCH`)** on APIs and internal libraries.
- Verify API & event compatibility in CI with **Pact (`can-i-deploy`)**, **`openapi-diff`**, and **Schema Registry compatibility checks** before merging.

---

#### 9. Service Lifecycle Management

End-to-end cradle-to-grave governance of a microservice:
1. **Bootstrap:** Golden templates / Internal Developer Portal (**Backstage / Red Hat Developer Hub**) pre-wired with CI/CD, OpenTelemetry, Actuator, Dockerfile, and Helm charts.
2. **Operate & Evolve:** GitOps (**ArgoCD / Flux**), SLO error budgets, automated dependency/CVE patching.
3. **Deprecate & Retire:** Announce via `Deprecation` + `Sunset` HTTP headers, track caller traffic in Gateway metrics down to `0`, archive DB/Kafka data per retention policy, delete K8s manifests, and revoke Vault/IAM credentials.

---

### 11.4 Twelve-Factor Applications

The **Twelve-Factor App** methodology defines best practices for portable, resilient, cloud-native microservices:

| # | Factor | Core Principle | Cloud-Native Java / Kubernetes Practice |
| --- | --- | --- | --- |
| **1** | **Codebase** | One codebase tracked in revision control, many deploys | One Git repo per microservice (or monorepo module) deployed via immutable tags across `dev`, `staging`, `prod` |
| **2** | **Dependencies** | Explicitly declare and isolate dependencies | Locked `pom.xml`/`build.gradle` + Maven Wrapper (`mvnw`) + self-contained multi-stage Docker image |
| **3** | **Config** | Store config that varies across deploys in the environment | Environment variables, K8s `ConfigMaps`, `Secrets`, and Vault (zero hardcoded prod IPs/passwords in Git!) |
| **4** | **Backing Services** | Treat DBs, caches, and brokers as attached resources | Swap Testcontainers PostgreSQL/Kafka in test ↔ AWS RDS / Confluent Cloud in prod via URL/config alone |
| **5** | **Build, Release, Run** | Strictly separate build, release, and run stages | CI builds immutable image (`Build`), Helm/ArgoCD binds config (`Release`), K8s runs pods (`Run`); never hot-patch a running pod! |
| **6** | **Processes** | Execute app as stateless, share-nothing processes | No sticky local sessions or local disk state; push state to PostgreSQL / Redis / S3 |
| **7** | **Port Binding** | Export services via self-contained port binding | Embedded Tomcat / Netty / Vert.x listening on `:8080` directly inside the container (no external WebLogic `.war` deploy!) |
| **8** | **Concurrency** | Scale out via the process model | Scale horizontally with K8s HPA replicas across pods instead of scaling one giant JVM vertically |
| **9** | **Disposability** | Maximize robustness with fast startup & graceful shutdown | Quarkus / GraalVM / AppCDS fast startup + `preStop` + `server.shutdown=graceful` on `SIGTERM` |
| **10** | **Dev/Prod Parity** | Keep dev, staging, and prod as similar as possible | Use **Testcontainers / Quarkus Dev Services** (real Postgres + Kafka locally) instead of H2; frequent small CI/CD deploys |
| **11** | **Logs** | Treat logs as event streams written to `stdout`/`stderr` | Emit single-line structured JSON to `stdout`; let Fluent Bit / OTel Collector ship & retain logs externally |
| **12** | **Admin Processes** | Run admin/management tasks as one-off processes | Execute Flyway migrations, backfills, or data repairs via Kubernetes **`Job`s** using the exact same release image & config |

---

### 11.5 Service Mesh

---

#### 1. What is a Service Mesh?

A **Service Mesh** is a dedicated, programmable infrastructure layer that handles **East–West service-to-service networking** transparently outside application code—providing **mTLS encryption**, cryptographic service identity, L7 routing/canary splits, retries/timeouts/circuit breaking, and unified RED telemetry across polyglot microservices (Java, Go, Python, Node).

---

#### 2. Data Plane

The **Data Plane** consists of the lightweight, high-performance proxies (**Envoy** in Istio, **`linkerd2-proxy`** in Linkerd) deployed alongside workloads that **actually intercept, inspect, encrypt, load-balance, and forward every live network packet** between services.

---

#### 3. Control Plane

The **Control Plane** (`istiod` in Istio, Linkerd Control Plane) is the centralized management brain—**it NEVER touches live user request packets**:
- Watches Kubernetes Services/Endpoints and Mesh CRDs (`VirtualService`, `AuthorizationPolicy`).
- Acts as the internal **Certificate Authority (CA)** minting and rotating short-lived SPIFFE X.509 mTLS certificates.
- Pushes routing, TLS, and policy updates dynamically to all Data Plane proxies over **xDS** gRPC streams.

---

#### 4. Sidecar Proxy

In the classic **Sidecar Proxy** model, the Service Mesh control plane automatically injects a proxy container (e.g., `istio-proxy` / Envoy) into every application Pod:
- Pod startup `initContainer` configures `iptables` (or eBPF) rules so **all inbound and outbound TCP traffic seamlessly passes through the local `Envoy` sidecar via `localhost`**—the Java app thinks it's making plain HTTP calls to `http://payment-service:8080`, while Envoy upgrades it to **mTLS**, load-balances across pods via HTTP/2, records Prometheus metrics, and emits spans!

```text
┌─────────────────── Pod A (Order) ──────────────────┐       ┌────────────────── Pod B (Payment) ─────────────────┐
│ ┌─────────────┐  plain HTTP   ┌──────────────────┐ │ mTLS  │ ┌──────────────────┐  plain HTTP  ┌────────────┐ │
│ │ Order App   ├──────────────►│   Envoy Sidecar  ├─┼──────►│ │   Envoy Sidecar  ├─────────────►│ Payment App│ │
│ │ (Java/8080) │               │ (xDS, Cert, RED) │ │ HTTP/2│ │ (AuthZ, mTLS)    │              │ (Java/8080)│ │
│ └─────────────┘               └──────────────────┘ │       │ └──────────────────┘              └────────────┘ │
└────────────────────────────────────────────────────┘       └────────────────────────────────────────────────────┘
                                          ▲                                ▲
                                          └─────── Control Plane (istiod) ─┘ (xDS Config + SPIFFE Certs)
```

*(**Modern Sidecarless Evolution — Istio Ambient Mesh / Cilium eBPF:** Moves L4 mTLS to a shared node-level daemon `ztunnel` and optional L7 features to shared Waypoint proxies, slashing per-pod sidecar RAM/CPU overhead!)*

---

#### 5. Istio

**Istio** (CNCF graduated; upstream foundation of **Red Hat OpenShift Service Mesh**) uses **Envoy** (C++) proxies + **`istiod`**, offering deep L7 traffic & security controls via Kubernetes CRDs:
- **`VirtualService`:** Canary splits, header routing, timeouts, retries, fault injection (`abort` / `delay`), traffic mirroring.
- **`DestinationRule`:** Load-balancing algorithm (`LEAST_REQUEST`), TLS mode, connection pool limits, **outlier detection** (passive circuit breaking).
- **`PeerAuthentication`:** Enforce `STRICT` mTLS cluster-wide or per namespace.
- **`AuthorizationPolicy`:** Fine-grained L7 allow/deny rules (`ALLOW order-service SPIFFE ID to POST /payments`).

---

#### 6. Linkerd

**Linkerd** (CNCF graduated) prioritizes **simplicity, minimal resource footprint, and operational ease**, built around a custom ultra-fast **Rust micro-proxy (`linkerd2-proxy`)**:

| Dimension | Istio | Linkerd |
| --- | --- | --- |
| **Data Plane Proxy** | **Envoy** (C++, rich extensible ecosystem) | **`linkerd2-proxy`** (purpose-built Rust micro-proxy) |
| **Resource Overhead & Latency** | Higher RAM/CPU per sidecar (mitigated by Ambient mode) | **Ultra-low RAM/CPU** & minimal tail latency |
| **Feature Breadth** | Very deep (WASM plugins, complex egress/ingress gateways, multi-cluster, OpenShift native) | Focused essentials (automatic mTLS, gRPC/HTTP L7 LB, golden RED metrics, TrafficSplit, ServerAuthorization) |
| **Complexity** | Higher learning curve & CRD surface | Simple out-of-the-box defaults |

---

#### 7. Traffic Management

Service Mesh L7 capabilities without changing app code:
- **Canary & A/B Splits:** `95% → v1`, `5% → v2`, or route beta header `x-canary: true`.
- **Traffic Mirroring (Shadowing):** Fork a live copy of production requests asynchronously to `v2` (responses discarded) to test `v2` under real prod load with **zero user impact**!
- **L7 Load Balancing for HTTP/2 & gRPC:** Fixes Kubernetes L4 `ClusterIP` connection pinning cleanly.
- **Fault Injection:** Inject `500ms` latency or `5%` HTTP `503` errors selectively for specific test headers during Chaos GameDays.

---

#### 8. Service-to-Service Security

- **Automatic `STRICT` mTLS:** Encrypts 100% of East–West traffic automatically with 24h auto-rotated SPIFFE certs.
- **Zero-Trust L7 Authorization:** Define declarative `AuthorizationPolicy` restricting both *who* (`ServiceAccount` / SPIFFE ID) and *what* (`GET /public/**` vs `POST /payments`) each service may invoke—plus optional JWT validation right in the Envoy proxy!

---

#### 9. Observability

Every mesh proxy automatically emits uniform telemetry across **all languages** (even legacy containers you can't recompile!):
- **Uniform RED Metrics:** Request rate, error rate, and P50/P95/P99 latency histograms per `source_workload → destination_workload` exported to Prometheus.
- **Live Topology Graphs:** Tools like **Kiali** (Istio / OpenShift) visualize real-time service traffic, mTLS lock icons, and red/amber error edges automatically.
- **Distributed Tracing Caveat:** Envoy emits ingress/egress spans automatically, **but your Java app MUST still copy incoming `traceparent` headers onto outbound HTTP calls**—otherwise Envoy cannot link an inbound span on Service A to its outbound span toward Service B!

---

#### 10. Service Mesh versus API Gateway

| Dimension | API Gateway | Service Mesh |
| --- | --- | --- |
| **Traffic Direction** | **North–South** (External Clients → Edge → Cluster) | **East–West** (Service-to-Service *inside* Cluster) |
| **Primary Responsibilities** | External OAuth2/OIDC auth, API keys, per-tenant monetization & rate limiting, WAF, public API versioning, BFF transformation | Transparent **mTLS**, zero-trust internal AuthZ, L7 gRPC/HTTP2 load balancing, canary routing, outlier detection, uniform internal telemetry |
| **Interaction Warning** | **Watch Out for Double Retries!** Never configure 3 retries in Spring `Resilience4j` **AND** 3 retries in Istio `VirtualService` simultaneously ($3 \times 3 = 9\times$ retry amplification!)—choose one layer to own retries! |

---

#### Quick Interview Revision Checklist (Section 11)
- Can you explain **Linux Namespaces vs cgroups**, **PID 1 exec-form `ENTRYPOINT`**, **Multi-stage Dockerfiles** (Distroless / Red Hat UBI), and **JVM `-XX:MaxRAMPercentage=70.0`**?
- Can you walk through the **Kubernetes Pod Termination Race Condition** and write out the 4 pieces of **Zero-Downtime Graceful Shutdown** (`maxUnavailable: 0` + `readinessProbe` + `preStop: sleep 10` + `server.shutdown=graceful` + `terminationGracePeriodSeconds: 45`)?
- Can you compare **Rolling vs Blue-Green vs Canary** deployments and explain **Expand-and-Contract** zero-downtime DB migrations?
- Can you recite the key **Twelve-Factor App** principles (especially Config, Stateless Processes, Port Binding, Disposability, Dev/Prod Parity, `stdout` Logs, and Admin `Job`s)?
- Can you explain **Service Mesh Data Plane vs Control Plane**, **Sidecar proxies** (and Ambient Mesh), **Istio vs Linkerd**, **Traffic Mirroring**, **SPIFFE mTLS**, and **why you must avoid double-retrying between Resilience4j and Istio**?
