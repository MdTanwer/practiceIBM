# Microservices Study Notes — Section 6

## 6. Idempotency and Message Reliability

---

### 6.1 Idempotency

---

#### 1. What is Idempotency?

Mathematically, an operation $f$ is idempotent if applying it multiple times produces the exact same result as applying it once: $f(f(x)) = f(x)$.

In distributed microservices, **idempotency** means executing an API call or message handler **1 time or $N$ times produces the exact same final server/database state** without unintended duplicate side effects (such as charging a customer twice or deducting stock twice).

##### Crucial Nuance: Server State vs HTTP Response
- **Native REST Idempotency** guarantees **identical server state**, not necessarily identical status codes:
  - First `DELETE /orders/ord-5` → deletes order, returns `204 No Content`.
  - Second `DELETE /orders/ord-5` → order is already deleted, returns `404 Not Found` or `204 No Content`. Either way, server state is identical!
- **Explicit `Idempotency-Key` Idempotency** (e.g., Stripe-style `POST /payments`) guarantees **both identical server state AND replaying the exact cached response** (`201 Created`, same `paymentId`) to the retrying client.

---

#### 2. Idempotent HTTP Methods

| Method | Safe? | Idempotent? | Nuance / Production Gotcha |
| --- | --- | --- | --- |
| **`GET`** | Yes | **Yes** | Read-only; never trigger side effects (like incrementing counters/mutating state) inside a `GET`! |
| **`HEAD`** | Yes | **Yes** | Same as `GET`, headers only. |
| **`OPTIONS`** | Yes | **Yes** | Read-only CORS / capability discovery. |
| **`PUT`** | No | **Yes** | Replaces the entire resource at `/users/42` with the exact payload. Sending it 5 times leaves the exact same resource state. |
| **`DELETE`** | No | **Yes** | Deleting `/users/42` 1 time or 5 times results in `/users/42` not existing. |
| **`POST`** | No | **No** *(by default)* | Calling `POST /payments` twice creates 2 charges unless protected with an `Idempotency-Key`. |
| **`PATCH`** | No | **Depends!** | `{"status": "CANCELLED"}` is **idempotent**; `{"balanceDelta": -50}` is **NOT idempotent**! |

---

#### 3. Idempotent POST Operations

`POST` requests are the #1 source of duplicate transactions in production because networks drop responses and users double-click "Pay" or refresh pages:
- **Example:** Mobile app sends `POST /api/v1/payments` over shaky 4G → Server charges `$100` and returns `201 Created`, **but the TCP response drops before reaching the phone** → Mobile client retries `POST /api/v1/payments` → **Double charge ($200)!**

To make `POST` safe to retry:
1. **Explicit `Idempotency-Key` Header:** Client attaches a unique key (`Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd`) generated once per logical user action.
2. **Natural Business Unique Constraints:** Rely on domain uniqueness where natural keys exist (e.g., `UNIQUE(order_id)` on `invoices`, or `UNIQUE(cart_id, checkout_attempt_id)` on `orders`).

---

#### 4. Idempotency Keys

An **Idempotency Key** is a unique opaque token (usually a **UUIDv4** or deterministic composite key) generated **by the client** before sending a mutating request and reused identically across all retries of that same request.

##### Five Non-Negotiable Production Rules for Idempotency Keys
1. **Client-Generated, Never Server-Generated:** The client creates the key *before* the first attempt; generating it on the server fails if the first response is lost in transit.
2. **Generate a Fresh Key per Logical Action:** Clicking "Buy Now" creates `Key-1`; if `Key-1` times out, retries reuse `Key-1`. If the user later makes a *second separate purchase*, the UI must generate a new `Key-2`.
3. **Scope Keys to Tenant / User + Endpoint:** Always key storage by **`(user_id / tenant_id, http_method, path, idempotency_key)`** so Tenant A cannot accidentally collide with or read Tenant B's cached response!
4. **Validate Request Payload Fingerprint (`SHA-256` Hash):** Store a canonical hash of the request body alongside the key. If a client sends `Idempotency-Key: K1` with `amount = $100`, then mistakenly sends `Idempotency-Key: K1` with `amount = $500` → **reject with `422 Unprocessable Entity` or `409 Conflict`!**
5. **Only Cache Deterministic Outcomes (`2xx` and `4xx`):** If Request 1 fails transiently with `500` or `503` (DB temporarily down), **delete/roll back the idempotency record** so the client can safely retry with the same key!

---

#### 5. Request Deduplication

```text
Incoming POST /payments (Header: Idempotency-Key = K1, BodyHash = H1)
   │
   ▼
[ Atomic INSERT INTO idempotency_keys (key=K1, hash=H1, status='IN_PROGRESS') ON CONFLICT DO NOTHING ]
   │
   ├──► Inserted 1 row (Won lock!)
   │      ├── Execute business logic / DB write / Payment charge
   │      ├── Update record: status='COMPLETED', response_code=201, response_body=...
   │      └── Return 201 Created (or if 5xx crash/error: DELETE key K1 so retries can try again!)
   │
   └──► Inserted 0 rows (Duplicate detected! Read existing row for K1)
          ├── If existing.hash != H1        ──► Return 422 Unprocessable Entity (Payload mismatch!)
          ├── If status == 'IN_PROGRESS'   ──► Return 409 Conflict (or 425 Too Early + Retry-After: 1)
          └── If status == 'COMPLETED'     ──► Replay exact cached response_code + response_body!
```

---

#### 6. Idempotency Storage

Where should you store `idempotency_keys`?

| Storage Option | How It Works | Pros & Cons |
| --- | --- | --- |
| **Same Relational DB (`PostgreSQL / MySQL`)** *(Recommended for critical writes)* | Store `idempotency_keys` table in the **same DB** as `orders`/`payments` | **Atomic ACID commit!** Impossible for business data to commit without the idempotency key (and vice versa). Slightly higher DB write load. |
| **Redis (`SET key value NX PX`)** | Fast in-memory check before hitting service/DB | **Sub-millisecond latency**, native TTL expiry. **Danger:** Not atomic with PostgreSQL! If DB commits and Redis fails/evicts key, duplicates slip through. |
| **Hybrid (Redis Fast Lock + PostgreSQL Durable Record)** | Acquire short-lived Redis lock to reject concurrent storms early, persist final state + response atomically in PostgreSQL | Combines Redis speed against concurrent bursts with PostgreSQL ACID durability. |

##### Production PostgreSQL Schema
```sql
CREATE TABLE idempotency_keys (
    user_id         VARCHAR(64)  NOT NULL,
    idempotency_key VARCHAR(128) NOT NULL,
    request_hash    CHAR(64)     NOT NULL,      -- SHA-256 of method + path + canonical body
    status          VARCHAR(20)  NOT NULL,      -- IN_PROGRESS | COMPLETED
    response_code   INT,
    response_body   JSONB,
    locked_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ  NOT NULL,
    PRIMARY KEY (user_id, idempotency_key)
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys (expires_at);
```

---

#### 7. Idempotency Expiration

You cannot keep idempotency keys forever without bloating disk and B-tree indexes. Choose TTLs based on retry windows:
- **Synchronous User Payment / Checkout APIs:** **24 hours** (matches Stripe's default TTL—covers browser refreshes, mobile retries, and brief outages).
- **B2B / Partner Webhook & Async APIs:** **7 days** (partners may queue failed webhooks and retry over days).
- **Kafka Consumer `inbox_processed_events`:** Match or slightly exceed **Kafka topic retention window** (e.g., **7–30 days**) so full topic replays remain deduplicated.
- **Cleanup Strategy:** Use Redis `PX`/`EX` native TTL, batched PostgreSQL deletes (`DELETE ... WHERE expires_at < NOW() LIMIT 5000`), or **daily time-range table partitions** (`DROP PARTITION`).

---

#### 8. Concurrent Duplicate Requests

A **concurrent duplicate race condition** happens when a user double-clicks "Pay" (or two load-balanced mobile retries fire **5ms apart**), hitting **Pod A** and **Pod B** simultaneously *before* Request 1 has finished committing!

##### Why Naive `SELECT` → `INSERT` Fails
Under default PostgreSQL/MySQL **`READ COMMITTED`** isolation:
1. Pod A runs `SELECT * FROM idempotency_keys WHERE key = 'K1'` → **not found**.
2. Pod B runs `SELECT * FROM idempotency_keys WHERE key = 'K1'` → **not found**.
3. Both Pod A and Pod B charge the user's credit card!

##### Correct Solutions
1. **Atomic `INSERT ... ON CONFLICT DO NOTHING` (`PRIMARY KEY` Lock):**
   - Pod A runs `INSERT INTO idempotency_keys (..., status='IN_PROGRESS') ON CONFLICT DO NOTHING` → **1 row inserted** (proceeds).
   - Pod B runs the same `INSERT` → **0 rows inserted** (or blocks briefly if uncommitted in the same transaction until Pod A commits/rolls back, then sees `COMPLETED` or `IN_PROGRESS`) → immediately rejects with **`409 Conflict` / `425 Too Early`** (`Retry-After: 1`) or returns the cached `COMPLETED` response!
2. **Stale `IN_PROGRESS` Lock Recovery:**
   - What if Pod A crashes (`kill -9` / OOMKill) *after* committing `IN_PROGRESS` and *before* calling Stripe/marking `COMPLETED`?
   - Store `locked_at`: if an existing `IN_PROGRESS` row is older than the max request SLA (e.g., `locked_at < NOW() - INTERVAL '60 seconds'`), allow a conditional reclaim (`UPDATE idempotency_keys SET locked_at = NOW() WHERE ... AND status = 'IN_PROGRESS' AND locked_at < ...`) so the key doesn't stay stuck permanently!

---

#### 9. Payment Idempotency

Financial and payment systems (IBM Banking, Stripe, Adyen) require **multi-hop idempotency** spanning `Client → OrderService → PaymentService → External Bank/Stripe API`:

```text
Client ──(Idempotency-Key: K1)──► OrderService ──(payment_idempotency_key: "pay-" + orderId)──► PaymentService
                                                                                                      │
                                                                             (Forwards SAME key!)     ▼
                                                                                             External Payment Gateway
                                                                                                (Stripe / Card Network)
```

##### Handling the "Crash After Calling Stripe, Before Local DB Commit" Scenario (Recovery Points)
Remember **Section 5.1**: never hold a DB transaction open while calling Stripe over HTTP! Instead, use **3 short local DB steps (Recovery Points)**:
1. **Step 1 (Local ACID Tx 1):** Validate request, insert `idempotency_keys` (`IN_PROGRESS`) + insert `payment` row (`status = 'INITIATED'`, `gateway_key = UUID`). Commit!
2. **Step 2 (External HTTP Call — No DB Tx open):** Call Stripe `POST /v1/charges` with header **`Idempotency-Key: <gateway_key>`**.
   - If `PaymentService` pod crashes *right after Stripe charges the card* before Step 3, `payment` remains `INITIATED` in DB with the saved `gateway_key`.
   - When client retries with `K1` (or background reconciliation job picks up stale `INITIATED` payments), `PaymentService` calls Stripe again with the **exact same `gateway_key`** → Stripe returns the original `charge_id` **without charging the card a second time!**
3. **Step 3 (Local ACID Tx 2):** Update `payment.status = 'CAPTURED'`, insert `Outbox` `PaymentCapturedEvent`, and mark `idempotency_keys.status = 'COMPLETED'`. Commit!

---

#### 10. Idempotent Consumers

Asynchronous consumers (Kafka, RabbitMQ, SQS) inevitably receive duplicate messages. Build idempotency into every consumer using one of two patterns:

1. **Explicit Deduplication (Transactional Inbox):**
   - Store processed `event_id` in `inbox_processed_events` inside the **same local `@Transactional` DB commit** as the domain state update.
   - Universal—works even for append-only operations like `ledger_entries` or `balance += 50`!
2. **Natural / State-Machine Idempotency (Zero extra table!):**
   - **Conditional State Transitions:**
     ```sql
     UPDATE orders SET status = 'PAID', paid_at = NOW()
     WHERE id = :orderId AND status = 'PENDING_PAYMENT';
     -- Second delivery finds status = 'PAID' -> 0 rows updated -> safe no-op!
     ```
   - **Monotonic Version Guard (CQRS Projections):**
     ```sql
     INSERT INTO order_summary_view (order_id, status, version)
     VALUES (:id, :status, :version)
     ON CONFLICT (order_id) DO UPDATE
       SET status = EXCLUDED.status, version = EXCLUDED.version
     WHERE EXCLUDED.version > order_summary_view.version;
     ```
   - **Unique Business Constraint:** `UNIQUE(order_id)` on `shipments` prevents creating two shipments for the same order.

---

### 6.2 Message Delivery Guarantees

---

#### 1. At-Most-Once Delivery

A message is delivered **0 or 1 times**—**never duplicated, but may be silently lost**.
- **How it happens in Kafka:**
  - Producer sets `acks=0` and `retries=0` (fire-and-forget; never knows if broker crashed before persisting).
  - Or Consumer uses `enable.auto.commit=true` (or calls `ack()` *before* processing): if consumer crashes mid-processing, offset is already committed and the message is lost forever.
- **When acceptable:** Non-critical telemetry, high-volume clickstream logs, real-time stock ticker/IoT frames where a newer reading replaces the old one seconds later.

---

#### 2. At-Least-Once Delivery

A message is delivered **1 or more times**—**zero data loss, but duplicates can and will happen**.
- **How it is configured in Kafka:**
  - **Producer:** `acks=all` + `min.insync.replicas=2` (requires leader + ≥1 follower replica to flush before acking) + `retries=Integer.MAX_VALUE` + `enable.idempotence=true`.
  - **Consumer:** `enable.auto.commit=false` + manual offset commit **strictly AFTER** local DB transaction commits.
- **Why it is the Industry Default:** Losing a payment, order, or inventory update is unacceptable; handling duplicates safely in consumers is straightforward.

---

#### 3. Exactly-Once Semantics (EOS)

A message is processed **strictly 1 time**—no data loss and no duplicate side effects.

##### How Apache Kafka Implements EOS Internally
1. **Idempotent Producer (`enable.idempotence=true` — default since Kafka 3.0):**
   - Broker assigns each producer a `ProducerID (PID)` and each message a monotonic `SequenceNumber` per partition.
   - If the producer retries because a broker ACK timed out, the broker sees the duplicate `(PID, Partition, Seq)` and discards it cleanly.
2. **Kafka Transactions (`transactional.id` + `isolation.level=read_committed`):**
   - Atomically commits **both** produced output topic records **and** consumed input offsets (`sendOffsetsToTransaction`) inside Kafka's internal coordinator (`__transaction_state`)—ideal for **Kafka Streams / Kafka-to-Kafka** ETL pipelines.

> **Crucial Interview Trap:** Kafka's built-in EOS **ONLY works inside Kafka-to-Kafka pipelines!** It does **NOT** span an external PostgreSQL database, MongoDB, or Stripe REST API!

---

#### 4. Effectively-Once Processing

Because Kafka cannot wrap PostgreSQL in a single distributed transaction without slow 2PC, microservices achieve **Effectively-Once Processing** across Kafka + External Databases via:

$$\text{Effectively-Once Processing} = \text{At-Least-Once Delivery} + \text{Idempotent Consumer (Transactional Inbox / Natural Idempotency)}$$

| Guarantee | Data Loss? | Duplicates? | Performance | Typical Usage |
| --- | --- | --- | --- | --- |
| **At-Most-Once** | Possible | Never | Highest | Metrics, telemetry, disposable logs |
| **At-Least-Once** | Never | Possible | High | General messaging (when paired with idempotent consumer) |
| **Kafka EOS** | Never | Never *(Kafka-to-Kafka only)* | Moderate | Kafka Streams / Flink read-transform-write pipelines |
| **Effectively-Once** | Never | Deduplicated at DB! | High | **Orders, Payments, Inventory (Kafka + RDBMS)** |

---

#### 5. Duplicate Message Delivery

Why do duplicate messages occur in production even when code looks bug-free? **4 real-world causes:**
1. **Producer Retry Without Idempotence:** Broker persists message, network drops the `ACK`, producer retries (`enable.idempotence=false`) → 2 copies written to Kafka partition.
2. **Outbox Relay / Debezium Crash:** Debezium reads `outbox_events` and publishes `Event X` to Kafka, then crashes *before* saving its Kafka Connect source offset → replays `Event X` on restart.
3. **Consumer Crash After DB Commit, Before Offset Commit:**
   - Consumer processes `offset 42` and commits PostgreSQL transaction (`UPDATE inventory...`).
   - Pod is **OOMKilled** 2ms later before calling `ack.acknowledge()`!
   - Replacement pod restarts from committed `offset 41` and re-reads `offset 42`.
4. **Kafka Consumer Group Rebalance (`max.poll.interval.ms` Exceeded):**
   - Pod A polls 500 messages and takes 6 minutes processing them (exceeding default 5-min `max.poll.interval.ms`).
   - Broker assumes Pod A died, revokes Partition 0, and assigns it to **Pod B**.
   - Pod B re-reads uncommitted `offset 42` while Pod A finishes processing it (**both pods process `offset 42` simultaneously!**).

---

#### 6. Message Acknowledgement

**Acknowledgements (ACKs)** tell the broker when a consumer has safely finished handling a message:
- **RabbitMQ (`AMQP`):**
  - `basicAck`: Remove message from queue permanently.
  - `basicNack(requeue=true)`: Put back on queue (danger: infinite tight loop on poison pills!).
  - `basicNack(requeue=false)` / `basicReject`: Drop or route directly to **Dead-Letter Exchange (DLX)**.
- **Spring Kafka (`AckMode`):**
  - **`AckMode.MANUAL_IMMEDIATE`:** Gives explicit control (`Acknowledgment.acknowledge()`).
  - **Critical Ordering Rule (After-Commit Ack):** Never acknowledge inside a `@Transactional` method before `COMMIT` succeeds! If PostgreSQL rolls back on `COMMIT` (e.g., serialization error, constraint, connection drop), you already committed the Kafka offset and **lost the message**. Use `TransactionSynchronizationManager` `afterCommit` (or commit DB method first, then call `ack.acknowledge()` right after it returns!).

---

#### 7. Offset Management

Kafka does not delete messages on read; each partition is an append-only log tracked per `ConsumerGroup` in the internal **`__consumer_offsets`** topic:

```text
Partition 0:  [0] [1] [2] [3] [4] [5] [6] [7] [8]
                               ▲       ▲       ▲
                       Committed    Current   Log-End Offset (LEO)
                       Offset (4)  Position  (8)
                       └─────── Consumer Lag = LEO - Committed = 4 ───────┘
```

##### Offset Commit Modes Comparison
| Mode | Behavior | Risk / Trade-Off |
| --- | --- | --- |
| **`enable.auto.commit=true`** *(every 5s)* | Commits highest polled offset on background timer | **Dangerous:** Can lose messages if worker threads haven't finished yet, AND creates large duplicate windows on crash! |
| **Manual `commitSync()`** | Blocks until broker confirms offset commit | Safe; slightly higher latency per batch/record. |
| **Manual `commitAsync()` + Rebalance Hook** | Non-blocking commits during steady processing + `commitSync()` inside `ConsumerRebalanceListener.onPartitionsRevoked()` | High throughput AND clean handoff on K8s pod scale/rebalance! |
| **DB-Stored Offsets** | Store `(topic, partition, offset)` in a table inside the **exact same ACID DB transaction** as business write; on startup `consumer.seek(...)` to DB offset | **Zero duplicates** on consumer crash (requires single-partition sequential processing per thread). |

---

#### 8. Consumer Retries

When a Kafka consumer fails to process a message (e.g., downstream DB briefly unavailable or optimistic lock failure):

##### Strategy A: Blocking (In-Place) Retries
- Retry 2–3 times quickly on the same consumer thread (`500ms → 1s → 2s`).
- **Pros:** Preserves strict partition ordering.
- **Cons:** While retrying Message #10 for 30 seconds, Messages #11–#10,000 in that partition are **completely blocked**—and long retries trigger **`max.poll.interval.ms` rebalance loops**!

##### Strategy B: Non-Blocking Delayed Retry Topics (`@RetryableTopic` in Spring Kafka)
```text
[order-events] ──(fails)──► [order-events-retry-10s] ──(fails)──► [order-events-retry-1m] ──► [order-events-dlt]
     │
     └─► Immediately advances offset and keeps processing other healthy messages!
```
- **Pros:** Keeps main partition lag near zero; never triggers rebalance timeouts; supports long multi-minute/hour backoff.
- **Cons:** **Breaks strict per-key ordering**! (Order `ord-1` Event A goes to `retry-10s` while Event B arrives on main topic and executes first—protect with `aggregateVersion` checks!).

---

#### 9. Dead-Letter Queues (DLQ / DLT)

A **Dead-Letter Queue / Topic** (`<topic>.DLT` / `<queue>.dlq`) is a quarantine destination for messages that fail permanently or exhaust all retry attempts.

##### Production DLQ Best Practices
1. **Rich Diagnostic Headers:** Spring Kafka's `DeadLetterPublishingRecoverer` attaches `kafka_dlt-original-topic`, `kafka_dlt-original-partition`, `kafka_dlt-original-offset`, `kafka_dlt-exception-fqcn`, `kafka_dlt-exception-message`, and `kafka_dlt-exception-stacktrace`.
2. **Real-Time Alerting:** Monitor DLQ message rate/count in Prometheus/Grafana—**any message landing in a production DLQ must fire an alert**, otherwise failed orders/payments silently vanish into the DLQ!
3. **Replay Tooling:** Build an admin endpoint or CLI workflow (or Kafka UI / Kafdrop / custom consumer) to inspect, patch if needed, and republish fixed DLQ messages back to the main topic cleanly.

---

#### 10. Poison Messages

A **poison message (poison pill)** is a message that **deterministically fails on every processing attempt**, causing an infinite crash-and-redeliver loop that halts the entire consumer group:
- **Classic Causes:**
  1. Malformed JSON or incompatible Avro/Protobuf bytes causing **`DeserializationException`** before the `@KafkaListener` method even runs! (By default, the Kafka consumer thread throws on `poll()` infinitely without advancing offsets!)
  2. Unexpected `null` field causing `NullPointerException`.
  3. Unsatisfiable DB constraint (foreign key missing, string exceeds `VARCHAR(50)`).

- **Critical Fix in Spring Kafka (`ErrorHandlingDeserializer`):**
  - Wrap `JsonDeserializer` / `KafkaAvroDeserializer` inside **`ErrorHandlingDeserializer`** so bad bytes are caught, wrapped in a failed record header, and routed straight to the **DLQ** by `DefaultErrorHandler` instead of crashing the consumer forever!
  - Configure `DefaultErrorHandler.addNotRetryableExceptions(DeserializationException.class, ValidationException.class)` so poison pills go to DLQ **immediately on Attempt 1** without wasting retries.

---

#### 11. Message Ordering

##### Kafka's Fundamental Ordering Guarantee
Kafka guarantees strict message ordering **ONLY within a single partition**—never globally across multiple partitions of a topic!

- **How to keep entity events ordered:** Always pass **`aggregateId`** (e.g., `orderId` or `accountId`) as the **Kafka Message Key**. Default Murmur2 partitioning routes all events for `ord-778` (`OrderCreated` → `OrderPaid` → `OrderShipped`) to the **exact same partition** and single consumer instance.

##### 4 Hidden Ways Message Ordering Still Breaks!
1. **`max.in.flight.requests.per.connection > 1` without `enable.idempotence=true`:** Batch 1 fails and retries while Batch 2 succeeds first → out of order! (*Fix:* Keep `enable.idempotence=true`, which preserves ordering up to `max.in.flight = 5`!)
2. **Changing Topic Partition Count Later:** Increasing partitions from 6 to 12 changes `hash(orderId) % numPartitions`, routing new events for `ord-778` to a different partition while old events remain on the original partition!
3. **Non-Blocking Retry Topics:** Event 1 moves to `retry-10s`; Event 2 processes immediately on the main topic!
4. **Multi-Threaded Consumer Pools:** Polling a batch and handing records to an `ExecutorService` breaks partition order unless work is sharded deterministically by `key.hashCode() % workers`.

##### Consumer-Side Defense Against Out-of-Order Events
Always include a monotonic **`aggregateVersion`** (or sequence number `1, 2, 3...`) in every event:
- `if (event.version() <= currentEntity.getVersion())` → **ignore stale/duplicate older event**!
- `if (event.version() > currentEntity.getVersion() + 1)` → **gap detected** (wait/retry briefly for missing intermediate event `version - 1` before applying!).

---

#### 12. Event Replay

**Event Replay** rewinds consumer offsets (or replays an Event Store) to reprocess historical messages:
- `kafka-consumer-groups.sh --bootstrap-server ... --group billing-view --topic order-events --reset-offsets --to-datetime 2026-11-18T00:00:00.000 --execute` (must stop consumers in that group first!).

##### Production Safety Checklist for Event Replay
1. **Isolate Consumer Groups:** Prefer spinning up a **new temporary consumer group** (e.g., `order-summary-v2`) to rebuild a projection in parallel, then swap traffic—zero downtime for live traffic!
2. **Suppress External Side Effects:** Never trigger real credit card charges, SMS messages, or confirmation emails when replaying old events (separate pure read-projection services from side-effect services, or validate `event.occurredAt` / replay context).
3. **Guarantee Idempotency & Version Ordering:** Every state update during replay must be safely re-entrant (`Inbox` / `WHERE version < EXCLUDED.version`).
4. **Watch Backpressure & Lag Alerts:** Replaying 50M events hits DB connection pools and CPU hard—rate-limit batch sizes or temporarily scale up DB/pods.

---

#### 13. Deduplication Strategies

| Strategy | How It Works | Atomic with DB Write? | Memory / Storage Cost | Best Use Case |
| --- | --- | --- | --- | --- |
| **1. Transactional Inbox Table** | `INSERT INTO inbox_events (event_id) ON CONFLICT DO NOTHING` inside same ACID DB tx | **Yes (100% ACID)** | Moderate (B-tree index + TTL cleanup / partitions) | **Orders, Payments, Ledger, Inventory** (gold standard) |
| **2. Natural / State-Machine Idempotency** | `UPDATE orders SET status='PAID' WHERE id=? AND status='PENDING'` or `WHERE version < newVersion` | **Yes (100% ACID)** | **Zero extra table!** | Monotonic state transitions & CQRS read views |
| **3. Redis `SET NX EX`** | Check & set `event:{id}` with TTL in Redis before processing | **No** (risks split-brain if Redis crashes/evicts after DB commits) | Low RAM (TTL auto-evicts) | High-throughput notifications, caching, non-financial streams |
| **4. Kafka Streams / Flink State Store** | Windowed RocksDB key-value store keeping seen `eventId`s for $T$ minutes | **Yes** *(within Kafka EOS)* | Bounded by window size (disk-backed RocksDB) | Real-time stream deduplication before downstream consumption |
| **5. Bloom Filter + DB Fallback** | Check fast in-memory Bloom filter first; only query DB if Bloom says *"possibly seen"* | Paired with DB fallback | Tiny RAM (~bits per key) | Billions of high-speed events where >99% are unique |

---

#### Quick Interview Revision Checklist (Section 6)
- Can you distinguish **native HTTP idempotency (same server state)** from **explicit `Idempotency-Key` caching (same server state + replayed response)**?
- Can you explain **all 5 rules of `Idempotency-Key`** (client-generated, tenant-scoped, SHA-256 payload hash mismatch → `422`, `IN_PROGRESS` concurrent duplicate lock → `409`/`425`, do NOT cache transient `5xx` errors)?
- Can you walk through **multi-hop Payment Idempotency** (3 short DB transactions + recovery point with Stripe/Bank key) without holding a DB lock open during HTTP calls?
- Can you explain **why Kafka EOS (`enable.idempotence` + transactions) does NOT cover PostgreSQL**, and write out **Effectively-Once = At-Least-Once + Idempotent Consumer**?
- Can you list **4 concrete production causes of duplicate Kafka messages** and **4 ways message ordering breaks**?
- Can you explain **Poison Pills**, **`ErrorHandlingDeserializer`**, **Blocking vs Non-Blocking (`@RetryableTopic`) retries**, and **why you never `ack` Kafka inside `@Transactional` before commit**?
- Can you compare all **5 Deduplication Strategies** (Transactional Inbox, State-Machine, Redis `SET NX`, RocksDB window, Bloom Filter)?
