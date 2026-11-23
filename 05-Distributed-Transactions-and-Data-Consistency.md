# Microservices Study Notes — Section 5

## 5. Distributed Transactions and Data Consistency

> **Critical**

---

### 5.1 Distributed Transaction Fundamentals

---

#### 1. ACID Properties

In a single relational database, **ACID** guarantees reliable transaction execution:

| Property | Meaning | How the DB Achieves It |
| --- | --- | --- |
| **Atomicity** | **All-or-nothing:** Either every SQL statement in the transaction commits, or all changes roll back completely. | Undo / Write-Ahead Log (WAL) |
| **Consistency** | Database transitions from one valid state to another, satisfying all constraints (`NOT NULL`, `UNIQUE`, `FK`, `CHECK`, balances >= 0). | Constraint checks & triggers |
| **Isolation** | Concurrent transactions execute without interfering or exposing partial uncommitted state to each other. | MVCC (Multi-Version Concurrency Control) / Row Locks (`READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`) |
| **Durability** | Once `COMMIT` returns success, data survives power loss or crashes. | `fsync` to WAL before acknowledging commit |

---

#### 2. BASE Properties

Distributed microservices prioritize availability and partition tolerance over global ACID locking, following **BASE**:
- **Basically Available:** The system continues serving requests (possibly degraded or slightly stale) even if parts of the cluster fail or partition.
- **Soft State:** System state across services/caches may continue changing over time even without new user input (as async events propagate).
- **Eventually Consistent:** Once writes stop, all services and read replicas converge to a mutually consistent state after a finite delay.

| Dimension | ACID (Single DB) | BASE (Distributed Microservices) |
| --- | --- | --- |
| **Consistency Model** | Immediate strong consistency | **Eventual consistency** |
| **Coordination** | Pessimistic locks / MVCC in one engine | Asynchronous events, Sagas, compensating actions |
| **Availability & Latency** | Lower under cross-node partitions | **High availability & low latency** |
| **Isolation** | Guaranteed by DB isolation level | **No global isolation** (must design explicit semantic locks / invariants) |
| **Best For** | Within a single microservice's local DB | Across independent microservices & read projections |

---

#### 3. Local Transactions

A **local transaction** executes inside **one single microservice and its own single database** (`@Transactional` in Spring Boot or Quarkus JTA):
- Fast (sub-millisecond to low-millisecond), strongly isolated, and automatically rolled back by the database on runtime exceptions.
- **Golden Rule:** Use local ACID transactions inside each microservice boundary whenever updating one DDD Aggregate.

---

#### 4. Distributed Transactions

A **distributed transaction** is a single logical business workflow that requires atomic or coordinated state changes across **multiple independent databases, message brokers, or microservices**:
- Example: Placing an e-commerce order must:
  1. Create an `Order` in `OrderDB`
  2. Charge the customer in `PaymentDB` (or Stripe API)
  3. Deduct stock in `InventoryDB`
  4. Send confirmation via `NotificationService`

Because each service has its own private database, a single `@Transactional` annotation cannot roll back `PaymentDB` if `InventoryDB` fails!

---

#### 5. Transaction Boundaries

A **transaction boundary** defines the exact scope of an atomic commit.

##### Two Critical Rules for Microservice Transaction Boundaries
1. **Align with DDD Aggregates:** Keep local ACID transactions scoped to **one Aggregate inside one microservice** (e.g., `Order` + `OrderLineItems`).
2. **Never Hold a DB Transaction Open Across Network Calls!**
   - **Anti-Pattern:** Starting `@Transactional`, calling `restClient.post("/payments")` over HTTP (which could take 2–10 seconds or time out!), and then saving to DB.
   - **Why it destroys production:** Holds a precious HikariCP database connection and row locks hostage for seconds per request—exhausting the DB connection pool under modest load!
   - **Fix:** Complete remote HTTP calls *outside* the DB transaction, or execute local DB work + **Transactional Outbox** insert in one fast <10ms transaction and process remote steps asynchronously.

---

#### 6. Database per Service

In microservices, **each service owns its private data store** and no external service may read or write its tables directly:
- Preserves schema autonomy, independent scaling, fault isolation, and polyglot persistence (PostgreSQL for orders, MongoDB for catalog, Redis for sessions).
- Trade-off: Eliminates distributed foreign keys, cross-service `JOIN`s, and shared ACID transactions—forcing explicit distributed transaction and read-composition patterns.

---

#### 7. Shared Database Problems

Sharing one database across multiple services just to keep `@Transactional` joins creates fatal coupling:
1. **Schema Coupling:** A column rename or migration in `OrderService` breaks `ShippingService` in production.
2. **Bypassed Domain Invariants:** Another service mutating `orders` directly via SQL skips validation logic and forgets to publish `OrderUpdatedEvent`.
3. **Lock Contention & Deadlocks:** Long-running analytical queries or batch updates from one service lock rows or starve connection pools needed by checkout.
4. **No Independent Scaling or Tech Choice:** Every service stays pinned to one DB engine and bottleneck.

---

#### 8. Strong Consistency

**Strong consistency (Linearizability)** guarantees that **once a write commits, every subsequent read immediately sees that latest write**, across all nodes/clients:
- **Necessary for:** Financial account balance debits (preventing overdraft/double-spend), reserving the *last* concert ticket or airplane seat, authentication token revocation.
- **Cost:** Requires synchronous quorum writes or single-leader coordination—higher write latency and reduced availability if a network partition isolates the leader.

---

#### 9. Eventual Consistency

**Eventual consistency** guarantees that if no new updates are made, all replicas and downstream materialized views will converge to the same state over time (typically milliseconds to a few seconds):
- **Acceptable for:** Product catalog search indexes, order history read dashboards, customer profile caches, loyalty points balance, social feeds, analytics.
- **Handling UX Lag:**
  - **Read-Your-Own-Writes:** After a user updates their profile, pass a version token or read from the write service/session cache briefly so they don't see stale data on refresh.
  - **Explicit Intermediate States:** Return `202 Accepted` with `"status": "PROCESSING"` and update UI via polling, SSE, or WebSockets.

---

#### 10. Consistency Trade-Offs

| Criteria | Strong Consistency | Eventual Consistency |
| --- | --- | --- |
| **Read Freshness** | Always latest committed value | May lag behind recent writes briefly |
| **Write Latency** | Higher (synchronous quorum / locking) | Lower (local commit + async propagation) |
| **Availability During Partition** | Minority side rejects writes/reads (**CP**) | All reachable nodes keep serving (**AP**) |
| **Throughput & Scalability** | Limited by lock contention & coordination | Scales horizontally across services & regions |
| **Application Complexity** | Simpler reasoning (serializable view) | Must handle stale reads, conflicts, & sagas |

---

#### 11. CAP Theorem (and PACELC)

Formulated by Eric Brewer, the **CAP Theorem** states that a distributed data store can simultaneously guarantee at most **two** of the following three properties:
- **C — Consistency (Linearizability):** Every read receives the most recent write or an error.
- **A — Availability:** Every non-failing node returns a valid non-error response (without guaranteeing latest write).
- **P — Partition Tolerance:** The system continues operating even when network messages between nodes are dropped or delayed.

##### Why "P" Is Mandatory
Real networks **always** experience packet loss, switch failures, and fiber cuts—so you **cannot choose "CA"** in a distributed system. When a partition occurs, you must choose between:
- **CP (Consistency over Availability):** Reject requests on the minority side of the split to prevent conflicting writes or stale reads (e.g., **etcd**, **ZooKeeper**, **Consul**, single-leader PostgreSQL with synchronous replication, banking ledgers).
- **AP (Availability over Consistency):** Keep accepting reads and writes on both sides of the partition and reconcile conflicts afterward (e.g., **Cassandra**, **DynamoDB** default reads, **Netflix Eureka**, shopping carts).

##### PACELC Theorem (Extension by Daniel Abadi)
- **If Partition (P):** Trade off **A**vailability vs **C**onsistency.
- **Else (E — normal operation with NO partition):** Trade off **L**atency vs **C**onsistency (replicating synchronously to followers raises latency for strong consistency; replicating asynchronously keeps latency low with eventual consistency).

---

#### 12. Distributed Transaction Challenges

Why distributed transactions are fundamentally harder than local DB transactions:
1. **No Global Isolation (`I`):** Without 2PC locking, an intermediate state of Saga A (`Order = PENDING_PAYMENT`) is visible to concurrent queries or Saga B, risking **dirty reads** or **lost updates** (e.g., two concurrent orders reserving the last item unless guarded explicitly).
2. **Partial Failures & Ambiguity:** A network timeout means you don't know if the remote DB committed or rolled back.
3. **Heterogeneous Technology:** A workflow often spans **PostgreSQL + MongoDB + Kafka + Stripe REST API**—most of these do not support XA distributed transactions!
4. **No Automatic Rollback:** Once `PaymentService` commits a charge, a later error in `ShippingService` cannot "DB rollback" `PaymentDB`; you must write explicit **compensating business logic** (refunds).
5. **Clock Skew:** Servers do not share a single physical clock; wall-clock timestamps cannot safely order distributed transactions.

---

### 5.2 Transaction Coordination

---

#### 1. Two-Phase Commit (2PC)

**Two-Phase Commit (2PC)** (standardized in Java as **XA / JTA**) is a synchronous blocking protocol coordinated by a central **Transaction Coordinator** across multiple **Participants** (databases/brokers) to achieve **atomic all-or-nothing commits**.

```text
Coordinator                        Participant A (OrderDB)          Participant B (PaymentDB)
     │                                       │                                │
     │─────── 1. PREPARE ───────────────────►│                                │
     │─────── 1. PREPARE ────────────────────┼───────────────────────────────►│
     │        [Executes locally, writes WAL, │                                │
     │         ACQUIRES & HOLDS LOCKS!]      │                                │
     │◄────── 2. VOTE_COMMIT (Yes) ──────────│                                │
     │◄────── 2. VOTE_COMMIT (Yes) ──────────┼────────────────────────────────│
     │                                       │                                │
     │ [Writes COMMIT to durable log]        │                                │
     │─────── 3. COMMIT ────────────────────►│                                │
     │─────── 3. COMMIT ─────────────────────┼───────────────────────────────►│
     │        [Releases locks]               │                                │
     │◄────── 4. ACK ────────────────────────│◄────── 4. ACK ─────────────────│
```

---

#### 2. Prepare Phase

**Phase 1 (Voting Phase):**
1. Coordinator sends `PREPARE` to every participating resource manager.
2. Each participant executes the local SQL/work, writes changes durably to its **Write-Ahead Log (WAL)** so it can neither forget nor fail to commit if asked, and **acquires and holds exclusive row/table locks**.
3. If successful, participant replies **`VOTE_COMMIT`** (a binding promise that it *can* and *will* commit if told, even after a crash!). If any constraint or lock fails, it replies **`VOTE_ABORT`**.

---

#### 3. Commit Phase

**Phase 2 (Decision Phase):**
1. **If ALL participants voted `VOTE_COMMIT`:**
   - Coordinator durably writes `COMMIT` to its own transaction log (point of no return!).
   - Coordinator sends `COMMIT` to all participants. Participants finalize data, **release locks**, and reply `ACK`.
2. **If ANY participant voted `VOTE_ABORT` or timed out:**
   - Coordinator writes `ABORT` and sends `ROLLBACK` to all participants, who undo changes using their undo log and release locks.

---

#### 4. Coordinator Failure

What happens if the **Coordinator crashes** *after* receiving `VOTE_COMMIT` from participants in Phase 1, **before** sending the Phase 2 `COMMIT`/`ROLLBACK` decision?
- Participants are trapped in an **`INDOUBT` (uncertain) state**:
  - Participant A cannot safely abort (maybe the Coordinator already logged `COMMIT` and told Participant B to commit!).
  - Participant A cannot safely commit unilaterally (maybe Participant B voted `ABORT`!).
- Participants **must stay frozen holding all locks** until the Coordinator reboots and reads its recovery log!

---

#### 5. Blocking Behaviour

Because Phase 1 forces participants to **hold exclusive database locks across multiple network round-trips**—and indefinitely during a Coordinator outage—2PC is a **blocking protocol**:
- Every other unrelated request trying to read/update those locked rows stalls behind the indoubt XA transaction.
- Under high concurrency or network jitter, lock contention collapses throughput and triggers cascading connection-pool timeouts across the system.

---

#### 6. Limitations of 2PC

Why 2PC is **almost never used** in modern cloud-native microservices:
1. **Blocking on Coordinator Failure:** Locks remain held until Coordinator recovers.
2. **High Latency & Poor Scalability:** Requires multiple synchronous network RTTs + sequential forced disk `fsync`s on coordinator and every participant; speed is capped by the slowest participant.
3. **Incompatible with Modern Tech Stack:** REST/gRPC APIs, **Apache Kafka** (does not participate in external XA 2PC with PostgreSQL), MongoDB/Redis, and external SaaS APIs (Stripe, Twilio) do **not** implement XA/2PC!
4. **Reduces Availability:** If *any* 1 of 5 participants is unreachable, the entire transaction fails—violating microservices autonomy.

---

#### 7. Saga Pattern

The **Saga Pattern** replaces synchronous 2PC locking with a **sequence of independent local ACID transactions ($T_1, T_2, \dots, T_n$)** across microservices:
- Each local transaction $T_i$ commits immediately in its own DB (no long-lived distributed locks!) and triggers the next step via an event or command.
- If step $T_k$ fails, the Saga executes **Compensating Transactions ($C_{k-1}, \dots, C_1$)** in **reverse order** to semantically undo all previously committed steps.

##### Crucial Saga Property: ACD (No Isolation!)
Because each step $T_i$ commits immediately, **Sagas lack the `I` (Isolation) of ACID**:
- **Anomaly Example:** Order 1 reserves inventory (`qty: 1 → 0`), and before Order 1's payment finishes, Order 2 reads `qty = 0` and fails—even if Order 1's payment later fails and compensates stock back to `1`! Or a user views an order mid-flight while it is half-processed.
- **Countermeasures for Missing Isolation:**
  - **Semantic Lock:** Mark records with explicit transitional states (`ORDER_PENDING`, `BALANCE_FROZEN`) so concurrent workflows know a saga is in progress.
  - **Commutative Updates:** Design operations that produce the same result regardless of order (e.g., ledger debit/credit append entries rather than overwriting absolute balance).
  - **Pessimistic View Ordering / Re-read Validation:** Re-check preconditions at confirm step.

---

#### 8. Saga Choreography

In **Saga Choreography**, there is **no central coordinator**—each service listens for domain events, executes its local transaction (via Transactional Outbox), and emits the next event.

```text
Happy Path:
  OrderSvc ──[OrderCreated]──► PaymentSvc ──[PaymentCharged]──► InventorySvc ──[StockReserved]──► OrderSvc (COMPLETED)

Failure / Compensation Path (Stock Insufficient):
  InventorySvc ──[StockReservationFailed]──► PaymentSvc (Refunds!) ──[PaymentRefunded]──► OrderSvc (CANCELLED)
```

- **Pros:** Simple for small flows (2–4 steps), zero central SPOF, loosely coupled services.
- **Cons:** Workflow state is scattered across 5+ codebases; cyclic topic dependencies easily emerge (`OrderSvc ↔ PaymentSvc ↔ InventorySvc`); hard to answer *"Where is Order #778 stuck right now?"* or add timeouts across steps.

---

#### 9. Saga Orchestration

In **Saga Orchestration**, a dedicated **Saga Orchestrator** (e.g., **Temporal**, **Camunda / Zeebe**, **Axon**, or a persisted DB state machine in `OrderOrchestratorService`) owns the workflow state machine and sends explicit **Commands** to participants, listening for **Replies**:

```text
                       ┌──────────────────────────────────────┐
                       │    Order Saga Orchestrator (DB Log)  │
                       └──┬───────────────┬────────────────┬──┘
       1. Cmd: Reserve    │               │ 2. Cmd: Charge │ 3. Cmd: Ship
          Inventory       ▼               ▼                ▼
                    ┌───────────┐   ┌───────────┐   ┌─────────────┐
                    │ Inventory │   │  Payment  │   │  Shipping   │
                    │  Service  │   │  Service  │   │   Service   │
                    └─────┬─────┘   └─────┬─────┘   └─────────────┘
       Reply: OK/Fail ────┘               └── Reply: Fail! ──► Orchestrator sends
                                                               Cmd: ReleaseInventory!
```

| Dimension | Saga Choreography | Saga Orchestration |
| --- | --- | --- |
| **Coordination** | Decentralized (services react to events) | Centralized **Orchestrator** directs commands |
| **Workflow Visibility** | Scattered across services | **Clear single state machine** (`SagaInstance` table / Temporal UI) |
| **Coupling** | Participants must know which events trigger them | Participants stay dumb & decoupled; Orchestrator knows the flow |
| **Complex Logic / Timeouts** | Hard (timeouts & branching become messy) | **Easy** (built-in step timers, retries, conditional branches) |
| **Best For** | Simple 2–4 step workflows | **Complex workflows (4+ steps)**, financial/order pipelines, strict auditability |

---

#### 10. Compensating Transactions

A **Compensating Transaction ($C_i$)** is a **semantic, business-level undo operation**—**not** a physical database rollback!
- Once `PaymentService` commits a credit card charge or `EmailService` sends an email, you cannot execute SQL `ROLLBACK`.
- Compensation appends a **new forward record** that logically reverses the effect:
  - `ReserveStock(sku, 2)` ↔ `ReleaseStock(sku, 2)`
  - `DebitWallet($50)` ↔ `CreditWalletRefund($50)`
  - `CreateOrder` ↔ `MarkOrderCancelled(reason)` (audit trail preserved!)

##### Classifying Steps in Every Saga (Important Design Rule!)
1. **Compensatable Transactions (Before Pivot):** Steps that *can* be undone if a later step fails (`CreateOrder(PENDING)`, `ReserveInventory`, `AuthorizeCardHold`).
2. **Pivot Transaction (Point of No Return):** The decisive step whose success determines the Saga moves forward to completion (e.g., `CapturePayment` or `HandOffToCourier`).
3. **Retryable Transactions (After Pivot):** Steps occurring *after* the Pivot that **cannot fail permanently** and have no rollback—they are retried with backoff until they succeed (`ConfirmOrder`, `ScheduleEmail`, `GrantLoyaltyPoints`).

---

#### 11. Saga Failure Handling

- **Backward Recovery (Before Pivot):** If a step fails before the Pivot Transaction, execute compensating transactions $C_{k-1} \dots C_1$ in reverse order and mark the Saga `ABORTED / COMPENSATED`.
- **Forward Recovery (After Pivot):** Once the Pivot Transaction succeeds, **do not compensate backwards**—keep retrying remaining steps (with idempotency keys!) until the Saga finishes `COMPLETED`.
- **Crash-Safe Orchestrator:** Always persist every Saga state transition (`STEP_1_DONE`, `COMPENSATING_STEP_1`) in a durable database table (`saga_state`) using the **Transactional Outbox** so if the Orchestrator pod restarts mid-flight, another pod resumes cleanly.

---

#### 12. Saga Timeout

In asynchronous Sagas, a participant might crash or drop a reply message, leaving the Saga hanging forever in `WAITING_FOR_PAYMENT`.

##### How to Handle Saga Timeouts Safely
1. **Per-Step Deadline Timers:** Orchestrator persists `timeout_at = now + 30s` on each pending step; a scheduler detects expired steps and initiates compensation.
2. **Guard Against Late-Arriving Replies (Race Condition!):**
   - Suppose `PaymentService` took 35s; at 30s, the Orchestrator timed out, marked the Order `CANCELLED`, and released inventory. At 35s, `PaymentSucceeded` finally arrives!
   - **Rule:** Every Saga participant & orchestrator must check current Saga state—if a late `PaymentSucceeded` arrives for an already `CANCELLED` Saga, the Orchestrator **must immediately issue an automatic `RefundPayment` compensation command** (never silently ignore it!).

---

#### 13. Compensation Failure

What if a compensating transaction itself fails (e.g., `InventoryService` or `StripeRefundAPI` returns `503 Service Unavailable`)?

> **Ironclad Rule:** **Compensating transactions MUST be idempotent and retried indefinitely (with exponential backoff) until they succeed.** You cannot "compensate a failed compensation" by ignoring it—that leaks money or locks stock forever!

- **Automated Recovery Loop:** Retry locally with backoff → route to delayed retry topics (`5s → 1m → 15m → 1h`).
- **Human-in-the-Loop Escalation:** After $N$ automated attempts over several hours (permanent data/business error), route to a **Compensation Dead-Letter Queue (`saga-compensation.DLQ`)**, trigger a high-severity PagerDuty/Grafana alert (`CRITICAL`), and expose an ops reconciliation dashboard to replay/resolve manually.

---

#### 14. Try–Confirm–Cancel (TCC)

**TCC (Try–Confirm–Cancel)** is a high-consistency **2-phase application-level Saga** that eliminates dirty reads by explicitly **reserving/freezing** resources first:

1. **Try (Reserve Phase):** Check business constraints and **freeze** required resources in each service:
   - `AccountService`: move `$100` from `available_balance` → `frozen_balance`
   - `InventoryService`: move `2 units` from `available_qty` → `reserved_qty`
2. **Confirm (Execute Phase — if ALL `Try` calls succeed):** Convert frozen resources into final committed deductions (`frozen_balance -= $100`, `reserved_qty -= 2`). Must be **idempotent** and retried until success.
3. **Cancel (Release Phase — if ANY `Try` fails or times out):** Move frozen resources back (`frozen_balance -= $100; available_balance += $100`). Must be **idempotent** and retried until success (also attach a **reservation TTL** in case the coordinator dies permanently!).

##### Master Comparison: 2PC vs Saga vs TCC
| Dimension | 2PC (XA) | Standard Saga | TCC (Try–Confirm–Cancel) |
| --- | --- | --- | --- |
| **Locking Mechanism** | Database row/table locks (blocking) | None (immediate local commit + semantic lock) | **Application-level resource freeze (`reserved`/`frozen`)** |
| **Isolation** | Full ACID | ACD only (intermediate state visible) | **Strong application isolation** (other users can't spend frozen balance!) |
| **Throughput & Scalability** | Low | **Highest** | High (short local locks only during Try/Confirm/Cancel) |
| **Implementation Effort** | Low (handled by XA driver) | Medium (compensations) | **High** (must write Try + Confirm + Cancel for every participant!) |
| **Best For** | Legacy monoliths / single vendor DBs | Standard e-commerce, travel, async workflows | **High-stakes FinTech, payments, wallets, scarce inventory** |

---

### 5.3 Data Consistency Patterns

---

#### 1. CQRS (Command Query Responsibility Segregation)

**CQRS** separates the **write path (Commands)** from the **read path (Queries)** using distinct models—and in microservices, frequently distinct databases optimized for each workload.

```text
Write Path (Strong Consistency):
  Client ─► Command API ─► Order Command Service ─► PostgreSQL (Write DB)
                                                         │
                                              Outbox + Kafka Events
                                                         │
Read Path (Fast Queries & Joins):                        ▼
  Client ─► Query API   ─► Order Query Service   ◄── Elastic / Read DB (Materialized View)
```

##### Why CQRS Matters in Microservices
- Traditional CRUD shares one normalized table model for both writes and reads, creating slow cross-service joins or N+1 queries on complex dashboards.
- Splitting writes and reads lets you scale reads 10x independently, index differently, and pre-join data from multiple microservices into flat read views.
- *Note:* Start simple (same DB, separate DTO/service classes) before introducing two separate databases + Kafka synchronization!

---

#### 2. Command Model

The **Command Model** handles all state mutations (`PlaceOrderCommand`, `CancelOrderCommand`, `UpdateStockCommand`):
- Enforces **DDD Aggregates and business invariants** with **strong consistency**.
- Uses normalized relational schemas (e.g., PostgreSQL) or an Event Store optimized for fast single-aggregate transactions.
- **Rule:** Commands should return minimal acknowledgement (`201`/`202` + `orderId`) rather than complex multi-service query DTOs.

---

#### 3. Query Model

The **Query Model** handles all reads (`GetOrderSummaryQuery`, `SearchCustomerOrdersQuery`) with **zero side effects**:
- Stores **denormalized, pre-joined read documents** tailored directly to UI screens in read-optimized stores (Elasticsearch for full-text search, MongoDB/PostgreSQL JSONB for order summaries, Redis for hot lookups).
- Updated asynchronously by subscribing to domain events from one or more microservices (**eventual consistency**).

---

#### 4. Event Sourcing

Instead of updating a row in place (`UPDATE accounts SET balance = 400 WHERE id = 1`), **Event Sourcing** stores **every state change as an immutable append-only sequence of domain events** in an **Event Store**:

```text
Event Stream for Account #101:
  [1] AccountOpened(initialBalance = 0)
  [2] MoneyDeposited(amount = 500)
  [3] MoneyWithdrawn(amount = 100)
  ──────────────────────────────────────► Fold / Reduce in memory ──► Current Balance = 400
```

- **Benefits:** Built-in **100% immutable audit log** (ideal for banking/compliance), **time-travel queries** (*"What was balance on Nov 1st?"*), and zero impedance mismatch between DB writes and event publishing (paired naturally with CQRS!).
- **Challenges & Snapshots:** Replaying 1,000,000 events per read is slow → periodically persist a **Snapshot** (e.g., every 100 events: `Snapshot(version=100, balance=400)`) and replay only events after version 100; schema evolution requires **Upcasters** since past events are immutable.

---

#### 5. Event Replay

**Event Replay** re-reads historical events from offset `0` (in Event Sourcing Event Store or long-retention Kafka topics):
- **Use Cases:**
  1. Rebuilding a corrupted or newly redesigned **CQRS Materialized View**.
  2. Onboarding a **brand-new microservice** (e.g., `FraudAnalyticsService`) with full historical context.
  3. Recomputing state after fixing a consumer calculation bug.
- **Two Mandatory Guards During Replay:**
  1. Consumers must be **idempotent** (skip or overwrite deterministically by `eventId`/version).
  2. **Suppress external side effects!** Replaying 2 years of `OrderPlacedEvent`s must **never** re-send 500,000 confirmation emails or re-charge credit cards (separate pure state-projection consumers from side-effect consumers, or check `isReplay`/historical timestamp guards).

---

#### 6. Materialized Views

A **Materialized View** in microservices is a **persisted, pre-computed read table/collection** maintained by an event consumer that merges data from multiple owning services:
- Example: `OrderDashboardView` listens to `OrderCreated` (`OrderService`), `CustomerRenamed` (`CustomerService`), `PaymentCaptured` (`PaymentService`), and `PackageShipped` (`ShippingService`), upserting a single flat table row so `GET /dashboard/orders` runs in `< 5ms` with zero runtime HTTP calls!

---

#### 7. API Composition

Recap comparing the two ways to read data across **Database per Service**:

| Dimension | API Composition | CQRS Materialized View |
| --- | --- | --- |
| **How It Works** | Query owning services over HTTP/gRPC in parallel & join in memory | Consume Kafka events into a dedicated denormalized read DB |
| **Data Freshness** | **Real-time** (strong/immediate) | **Eventually consistent** (ms–seconds replication lag) |
| **Complex Filter / Sort / Paginate** | **Poor** (cannot efficiently SQL-join & paginate 100k rows in memory!) | **Excellent** (native SQL/Elasticsearch indexes across all fields) |
| **Runtime Availability** | Vulnerable if any downstream service is down | **High autonomy** (query DB works even if write services are offline) |
| **Complexity** | Low (no extra DB or Kafka sync) | Higher (extra DB, projections, replay, eventual consistency) |

---

#### 8. Eventual Consistency

When read models update asynchronously via events, **replication lag** (`50ms – 2s`) occurs between a write committing and the read view reflecting it.

##### Practical Patterns to Hide Lag from Users
1. **Read-Your-Own-Writes (Session Consistency):** After a user clicks "Save Profile" or "Place Order", return the created/updated state directly in the write HTTP response (`201 Created`) so the UI updates optimistically without immediately querying the lagging read view—or pass a monotonically increasing `version` token so the query service waits briefly until `view.version >= expectedVersion`.
2. **Idempotent Upsert with Version Guard:** When updating a CQRS read view from out-of-order events, only apply updates if `incomingEvent.aggregateVersion > storedRow.lastAppliedVersion`!

---

#### 9. Data Synchronization

Keeping replicated reference data and projections aligned across services:
1. **Event-Driven Sync (Preferred):** Transactional Outbox + Kafka events (`Event-Carried State Transfer`).
2. **Change Data Capture (CDC):** Debezium tails the write DB transaction log directly into Kafka topics.
3. **Periodic Reconciliation Batch Jobs:** In financial/inventory systems, even event pipelines can experience edge-case drift or missed manual DB hotfixes—run scheduled hourly/nightly **reconciliation jobs** comparing source-of-truth totals (`PaymentDB`) against downstream projections (`LedgerDB` / `OrderDB`) and emitting repair alerts/events on mismatch.

---

#### 10. Conflict Resolution

When multiple clients or distributed nodes update the same logical record concurrently:

| Mechanism | How It Works | Pros & Cons |
| --- | --- | --- |
| **Optimistic Concurrency Control (OCC)** | Include `@Version` column (`UPDATE inventory SET qty = qty - 1, version = version + 1 WHERE sku = ? AND version = 5`). If `0` rows updated → return **`409 Conflict`** | **Zero read locks!** Industry standard in REST APIs & JPA; caller retries with fresh version. |
| **Pessimistic Locking** | `SELECT ... FOR UPDATE` locks row until local DB transaction commits | Prevents retries under extreme hot-row contention (e.g., last flash-sale item), but lowers concurrency. |
| **Last-Write-Wins (LWW)** | Overwrite based on wall-clock timestamp | Simple, but **silently loses concurrent updates** and breaks under server clock skew! |
| **Vector Clocks / Version Vectors** | Track causal `[NodeA:2, NodeB:1]` counters; detect true concurrent writes and ask app/user to merge | Never silently drops data (Amazon shopping cart paper); complex client merge logic. |
| **CRDTs (Conflict-Free Replicated Data Types)** | Mathematically commutative/associative structures (`G-Counter`, `PN-Counter`, `OR-Set`) | Merge automatically across multi-region active-active nodes with **zero coordination**. |

---

#### 11. Distributed Locking

Used when multiple instances/pods of a microservice must guarantee **mutual exclusion** across processes (e.g., ensuring only 1 pod runs a scheduled billing cron job, or serializing a single account/seat mutation across nodes).

##### Common Implementations
1. **Database Locks / ShedLock / Advisory Locks:**
   - `INSERT INTO locks ...` / `SELECT ... FOR UPDATE SKIP LOCKED` / PostgreSQL `pg_advisory_lock`.
   - Best when your service already uses that DB (no extra infrastructure!).
2. **Redis Distributed Lock (`SET resource_key unique_token NX PX 30000`):**
   - `NX` = set only if not exists; `PX` = TTL auto-expiry so a crashed pod doesn't hold the lock forever.
   - **Safe Unlock (Lua Script):** Always check `if redis.call("get", KEY) == ARGV[1] then return redis.call("del", KEY) else return 0 end` so Pod A never accidentally deletes Pod B's lock after TTL expiration!
   - **Redlock Caveat:** Multi-node Redis Redlock still faces criticism (Martin Kleppmann) regarding clock jumps and JVM GC pauses.
3. **CP Consensus Stores (etcd / ZooKeeper / Consul):** Use Raft/Zab leases for **Leader Election** and critical cluster coordination.

##### Why Leases/TTL Alone Are Unsafe → Fencing Tokens!
Suppose Pod A acquires a Redis lock (`TTL = 10s`), freezes for **15 seconds in a Stop-the-World JVM GC pause**, wakes up thinking it still holds the lock, and writes to the DB—**after** the lock expired and Pod B already acquired the lock!

```text
Pod A acquires Lock (token=33) ──► [15s GC Pause! Lock TTL expires]
Pod B acquires Lock (token=34) ──► Writes Storage (last_token=34 OK!)
Pod A wakes up! Sends stale write (token=33) ──► Storage REJECTS (33 < 34)!
```

- **Solution — Fencing Token:** The lock service issues a **monotonically increasing token** (`33`, `34`, ...) on every lock grant, and the downstream database/storage rejects any write whose token is lower than the highest seen token (`WHERE fencing_token < :newToken`).

---

### 5.4 Transactional Messaging

---

#### 1. Dual-Write Problem

The **Dual-Write Problem** occurs when a microservice attempts to update its **local database** AND publish a **message to a broker (Kafka/RabbitMQ)** in the same method without a distributed 2PC transaction across both systems.

```java
// ANTI-PATTERN: Dual-Write Bug!
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);               // System 1: PostgreSQL
    kafkaTemplate.send("orders", event);       // System 2: Apache Kafka (Separate network system!)
}
```

---

#### 2. Database Commit and Message Publishing Failure

No matter which order you call the two operations in, **one of two catastrophic inconsistency bugs occurs**:

| Naive Order | Failure Timing | Resulting Production Bug |
| --- | --- | --- |
| **1. DB Save → 2. Kafka Send** | Kafka broker is unreachable or Pod crashes/OOMKills **after DB commit**, *before* Kafka `send()` acks | **Missing Event:** Order exists in `OrderDB`, **but `OrderPlacedEvent` is NEVER emitted!** Payment & Shipping stay unaware forever. |
| **1. Kafka Send → 2. DB Save** | Kafka `send()` succeeds first, **then DB commit fails** (unique constraint, deadlock, timeout, rollback) | **Phantom / Ghost Event:** `PaymentService` consumes `OrderPlacedEvent` and charges the customer's card for an Order that **rolled back and does not exist!** |

*(Even calling `kafkaTemplate.send().get()` inside `@Transactional` before commit fails: if Kafka succeeds and `COMMIT` itself fails due to deferred constraints or DB failover, the phantom event was already sent!)*

---

#### 3. Transactional Outbox Pattern

The **Transactional Outbox Pattern** solves the dual-write problem completely using a **single local ACID database transaction**:
1. Inside `@Transactional`, the service writes the business entity (`orders`) **AND** inserts the serialized event into an **`outbox_events` table in the exact same database**.
2. Because both tables live in the same database, either **both commit atomically** or **both roll back atomically** (zero ghost events, zero lost events!).
3. An asynchronous **Outbox Relay** (Polling Publisher or **Debezium CDC**) reads newly committed `outbox_events` rows and publishes them to Kafka reliably.

```text
┌──────────────────────────────────────────────────────────┐
│                 OrderService (@Transactional)            │
│   1. INSERT INTO orders (...)                            │
│   2. INSERT INTO outbox_events (id, payload, ...)        │
└─────────────────────────────┬────────────────────────────┘
                              │ Single Local ACID Commit!
                              ▼
                  ┌───────────────────────┐
                  │   Order PostgreSQL    │
                  │  [orders] + [outbox]  │
                  └───────────┬───────────┘
                              │
             ┌────────────────┴────────────────┐
             │ Outbox Relay (Polling OR        │
             │ Debezium CDC Tailing WAL)       │
             └────────────────┬────────────────┘
                              ▼
                     [ Apache Kafka ] ──► Consumers (with Transactional Inbox)
```

---

#### 4. Outbox Table

Production-ready SQL schema for `outbox_events` (PostgreSQL):

```sql
CREATE TABLE outbox_events (
    id             UUID PRIMARY KEY,
    aggregate_type VARCHAR(100) NOT NULL,      -- e.g. 'Order' (can route topic)
    aggregate_id   VARCHAR(100) NOT NULL,      -- Kafka partition key! Preserves per-order ordering
    event_type     VARCHAR(100) NOT NULL,      -- e.g. 'OrderPlaced'
    payload        JSONB        NOT NULL,      -- Serialized event data
    headers        JSONB,                      -- W3C traceparent, correlationId, schemaVersion
    status         VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    published_at   TIMESTAMPTZ
);

CREATE INDEX idx_outbox_pending ON outbox_events (created_at) WHERE status = 'PENDING';
```

- **Crucial Detail (`aggregate_id` as Kafka Key):** Using `aggregate_id` (`orderId`) as the Kafka message key ensures all events for the same order (`OrderCreated` → `OrderPaid` → `OrderCancelled`) land in the **same Kafka partition** in strict chronological order!
- **Crucial Detail (`headers`):** Store W3C `traceparent` inside `outbox_events.headers` so distributed OpenTelemetry tracing connects seamlessly across the async outbox hop!

---

#### 5. Outbox Polling

In **Polling Publisher**, a background scheduler runs periodically (e.g., every `200ms–1000ms`):
```sql
SELECT id, aggregate_type, aggregate_id, event_type, payload, headers
FROM outbox_events
WHERE status = 'PENDING'
ORDER BY created_at ASC
LIMIT 100
FOR UPDATE SKIP LOCKED;
```
- Publishes batch to Kafka (`acks=all`), awaits broker ack, then updates `status = 'PUBLISHED', published_at = NOW()`.
- **Pros:** Zero extra infrastructure (no Kafka Connect/Debezium cluster needed); works on any database (PostgreSQL, MySQL, Oracle, SQL Server).
- **Cons:** Adds polling latency (`~200ms–1s`), adds constant `SELECT` queries to DB under idle/busy load, and multi-pod pollers must be careful with **`FOR UPDATE SKIP LOCKED`** or **ShedLock** so parallel pollers don't reorder events for the same `aggregate_id`.

---

#### 6. Change Data Capture (CDC)

**Change Data Capture (CDC)** (most commonly **Debezium** running on **Kafka Connect**) captures committed row inserts from `outbox_events` **without polling SQL queries**:
- Debezium provides a built-in **`Outbox Event Router` Single Message Transform (SMT)** that reads `outbox_events` inserts, extracts `aggregate_type` as the Kafka topic, `aggregate_id` as the Kafka key, `payload` as the message value, and `headers` as Kafka headers automatically!

---

#### 7. Transaction Log Tailing

Under the hood, CDC works via **Transaction Log Tailing**:
- Every relational DB already writes committed changes sequentially to a **write-ahead transaction log**:
  - **PostgreSQL:** Write-Ahead Log (**WAL**) via logical decoding (`pgoutput`)
  - **MySQL:** Binary Log (**Binlog**)
  - **SQL Server / Oracle:** Transaction Log / Redo Log (LogMiner / XStream)
- Debezium tails this binary log stream in near real time (`< 10–50ms`), seeing **only committed transactions** in exact commit order with **negligible impact on OLTP queries**.

| Criteria | Outbox Polling | CDC / Transaction Log Tailing (Debezium) |
| --- | --- | --- |
| **Infrastructure** | Simple (just app code / scheduler) | Requires **Kafka Connect + Debezium** + DB logical replication config |
| **End-to-End Latency** | Higher (polling interval, e.g., 500ms–2s) | **Very low (milliseconds)** |
| **Database Load** | Repeated polling `SELECT` + `UPDATE` writes | **Minimal** (reads sequential WAL stream, zero SQL polling) |
| **Ordering Guarantee** | Must coordinate concurrent pollers carefully | **Exact DB commit order** naturally preserved from WAL |

---

#### 8. Transactional Inbox Pattern

The **Transactional Inbox Pattern** is the **consumer-side partner** of the Outbox pattern. It prevents duplicate message processing atomically inside the consumer's local DB transaction:

```sql
CREATE TABLE inbox_processed_events (
    event_id     UUID PRIMARY KEY,
    event_type   VARCHAR(100) NOT NULL,
    processed_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
```

```java
@KafkaListener(topics = "order-events")
@Transactional
public void handleOrderPlaced(OrderPlacedEvent event, Acknowledgment ack) {
    // 1. Try inserting eventId atomically: INSERT INTO inbox_processed_events (...) ON CONFLICT DO NOTHING
    int inserted = inboxRepository.insertIfNotExists(event.eventId(), "OrderPlaced");
    if (inserted == 0) {
        ack.acknowledge(); // Already processed earlier! Safe duplicate no-op.
        return;
    }

    // 2. Apply domain state mutation inside the EXACT SAME ACID transaction
    inventoryService.reserveStock(event.orderId(), event.items());

    // 3. Acknowledge Kafka offset ONLY AFTER DB transaction commits!
    // (Use TransactionSynchronization.afterCommit or Spring Kafka Chained/AfterCommit ack)
}
```

- Because `inbox_processed_events` and `inventory` update inside the **same local ACID DB transaction**, it is **impossible** for an event to be processed twice OR recorded as processed without updating inventory!

---

#### 9. Duplicate Event Handling

Even with Transactional Outbox + Debezium, **duplicate events WILL happen**:
- If the Polling Publisher or Debezium connector pushes `eventId = X` to Kafka, Kafka persists it, and the relay pod **crashes before committing its WAL offset / updating `status = 'PUBLISHED'`**, upon restart the relay **re-publishes `eventId = X`** (**at-least-once delivery**).
- Combined with Kafka consumer rebalances before offset commit, **every consumer must assume duplicates will arrive** and deduplicate via **Transactional Inbox** (or natural idempotent SQL transitions).

---

#### 10. Outbox Cleanup

Unbounded growth of `outbox_events` and `inbox_processed_events` degrades index performance and causes heavy PostgreSQL `VACUUM` / dead-tuple bloat:
- **With CDC / Debezium (Clever Trick!):** Inside `@Transactional`, `INSERT` into `outbox_events` and optionally `DELETE` it right after (or in a quick cleanup job)—Debezium captures the `INSERT` record from the WAL and is configured to ignore `DELETE` tombstones, keeping the live `outbox_events` table near **zero rows**!
- **With Polling:** Run scheduled batched cleanup (`DELETE FROM outbox_events WHERE id IN (SELECT id FROM outbox_events WHERE status = 'PUBLISHED' AND published_at < NOW() - INTERVAL '7 days' LIMIT 5000)`) to avoid long table locks.
- **High-Throughput Best Practice:** **Time-Range Partitioning** (`outbox_events_2026_11_23`) — drop old partitions instantaneously (`DROP TABLE` / `DETACH PARTITION`) with **zero `VACUUM` bloat**. *(Keep `inbox_processed_events` retention aligned with Kafka topic retention + max replay window, e.g., 7–30 days.)*

---

#### 11. Event Publishing Guarantees

Putting all layers together yields **End-to-End Effectively-Once Processing** across microservices without 2PC:

```text
1. Producer DB Commit   ──► Local ACID Transaction (Entity + Outbox Row)
2. Relay / Debezium CDC ──► At-Least-Once Publish to Kafka (acks=all, enable.idempotence=true)
3. Consumer Processing  ──► Manual Ack (after DB commit) + Transactional Inbox (PK Deduplication)
──────────────────────────────────────────────────────────────────────────────────────────────────
   Result = Zero Lost Events + Zero Phantom Events + Zero Duplicate Side Effects = EFFECTIVELY-ONCE!
```

---

#### Quick Interview Revision Checklist (Section 5)
- Can you explain **ACID vs BASE**, **CAP vs PACELC**, and **why you must NEVER hold a DB `@Transactional` open across an HTTP call**?
- Can you walk through **2PC (Prepare & Commit phases)** and explain **why Coordinator failure causes blocking lock contention** and why Kafka/REST don't use XA?
- Can you contrast **Saga Choreography vs Saga Orchestration** and explain **Compensatable vs Pivot vs Retryable transactions**?
- How do you handle **missing Isolation (`I`) in Sagas**, **late-arriving Saga responses after timeout**, and **failed compensating transactions**?
- Can you compare **2PC vs Saga vs Try-Confirm-Cancel (TCC)** and explain when FinTech systems choose TCC?
- Can you contrast **CQRS + Materialized Views vs API Composition**, explain **Event Sourcing + Snapshots + Event Replay safety**, and describe **Fencing Tokens in Distributed Locking**?
- Can you draw the **Dual-Write Bug (both failure directions)** and walk through **Transactional Outbox (Polling vs Debezium WAL CDC) + Transactional Inbox + Outbox Cleanup** end-to-end?
