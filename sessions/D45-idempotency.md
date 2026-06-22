# D45 — Idempotency: Why It Matters, How to Implement It

---

## Why It Matters

In distributed systems, things fail and retry. Retries mean the same operation might execute twice. Idempotency makes operations safe to retry.

> **Real case: Razorpay/Stripe — double charging**
> User clicks "Pay" twice (network slow). Two POST requests. Both charge card. User charged ₹1,998 instead of ₹999. Fix: idempotency keys — server checks "already processed this key?" before charging.

> **Real case: Amazon — duplicate Kafka orders**
> Consumer reads `OrderPlaced`, creates order, crashes before offset commit (D37). Restarts, reads same event, creates order AGAIN. Fix: check `event_id` before processing.

---

## What Idempotency Means [CRITICAL]

Executing an operation multiple times = same result as executing once.

```
✅ Idempotent:     SET balance = 500    → run 10 times → balance = 500
❌ NOT idempotent: balance += 100       → run 10 times → balance = original + 1000
```

**Analogy:** Elevator button (idempotent — press 10 times, elevator called once) vs vending machine coin slot (not idempotent — each coin adds money).

---

## Where Retries Happen [CRITICAL]

| Retry source | Why | Example |
|---|---|---|
| **User** | Clicks button twice | "Pay" button clicked twice |
| **Browser/App** | Auto-retry on timeout | `fetch()` with retry |
| **API Gateway** | Upstream timeout | Nginx `proxy_next_upstream` (D07) |
| **Service-to-service** | HTTP/gRPC timeout | Orders retries Payment call |
| **Kafka consumer** | Crash before offset commit (D37) | Same event processed again |
| **Load balancer** | Backend unhealthy (D08) | Request sent to second server |

**You can't prevent retries. You CAN make operations safe to retry.**

---

## HTTP Methods and Idempotency (from D05) [CRITICAL]

| Method | Idempotent? | Why |
|---|---|---|
| **GET** | ✅ | Reading doesn't change anything |
| **PUT** | ✅ | "Set to THIS state" — same result every time |
| **DELETE** | ✅ | Already gone → same result |
| **POST** | ❌ | "Create new" — each call creates another |
| **PATCH** | Depends | `{name: "Aryan"}` ✅ vs `{balance: +100}` ❌ |

**The danger is POST.** Every POST to `/payments/charge` creates a new charge. Need idempotency keys.

---

## Idempotency Keys — The Standard Solution [CRITICAL]

Client generates a unique ID, sends with every request. Server deduplicates.

```
Request 1:
  POST /payments/charge
  Idempotency-Key: "idk_abc123"
  Body: {amount: 999}

  Server: key not processed → charge card → save key + result → return 200

Request 2 (retry, same key):
  POST /payments/charge
  Idempotency-Key: "idk_abc123"

  Server: key already processed → return saved result → card NOT charged again ✅
```

### Implementation

```python
def charge_payment(request):
    key = request.headers.get("Idempotency-Key")
    
    existing = db.query("SELECT result FROM idempotency_store WHERE key = %s", key)
    if existing:
        return Response(200, existing.result)  # cached, don't re-process
    
    receipt = payment_gateway.charge(request.body)
    
    db.execute("INSERT INTO idempotency_store (key, result) VALUES (%s, %s)", 
               key, json.dumps(receipt))
    
    return Response(200, receipt)
```

### Database Table

```sql
CREATE TABLE idempotency_store (
    key         VARCHAR(255) PRIMARY KEY,
    result      JSONB NOT NULL,
    created_at  TIMESTAMP NOT NULL
);
-- Unique constraint = automatic dedup
-- Clean up keys older than 24-48 hours
```

### Who Generates the Key?

**CLIENT generates.** Only client knows "I'm retrying the same request."

```
Common strategies:
  UUID v4:     "550e8400-e29b-41d4-..."        (random, unique)
  Business ID: "order_456_payment"              (meaningful)
  Hash:        sha256(user_id + order_id + amt) (deterministic)
```

---

## Database-Level Idempotency Patterns [CRITICAL]

### Pattern 1: Upsert (INSERT ON CONFLICT)

```sql
-- ❌ NOT idempotent
UPDATE accounts SET balance = balance + 100 WHERE user_id = 123;

-- ✅ Idempotent
INSERT INTO payments (payment_id, user_id, amount, status)
VALUES ('pay_abc', 123, 999, 'completed')
ON CONFLICT (payment_id) DO NOTHING;
```

### Pattern 2: Unique Constraint as Safety Net

```sql
CREATE TABLE order_events (
    event_id    VARCHAR(255) UNIQUE,  -- from Kafka event's event_id (D44)
    order_id    VARCHAR(255),
    event_type  VARCHAR(100),
    data        JSONB
);
-- Duplicate event_id → unique violation → skip
```

### Pattern 3: Conditional Update (Optimistic Locking)

```sql
UPDATE orders SET status = 'shipped', version = 5
WHERE order_id = 456 AND version = 4;

-- First run: version=4 → updates → 1 row ✅
-- Retry: version=5 now → 0 rows affected → no change ✅
```

---

## Idempotent Kafka Consumers (D37 + D44) [CRITICAL]

Kafka = at-least-once. Consumer MUST be idempotent.

```python
def process_event(event):
    event_id = event["event_id"]
    
    already = db.query("SELECT 1 FROM processed_events WHERE event_id = %s", event_id)
    if already:
        return  # skip duplicate
    
    with db.transaction():  # ATOMIC — both or neither
        create_order(event["data"])
        db.execute("INSERT INTO processed_events (event_id) VALUES (%s)", event_id)
    
    consumer.commit_offset()  # if crash here, event_id check saves us on retry
```

**Why transaction matters:**
```
❌ Without: create_order() ✅ → crash → processed_events not written → retry → DUPLICATE
✅ With:    create_order() + processed_events in ONE transaction → retry → event_id found → SKIP
```

---

## Full Picture — Idempotency at Every Layer [IMPORTANT]

```
User:           Idempotency-Key header (prevents double-click)
API Gateway:    Deduplicates by key
Order Service:  Upsert with order_id, publishes event with event_id
Kafka:          At-least-once (same event might arrive twice)
Payment Consumer: Checks event_id in processed_events
Payment Gateway:  Accepts idempotency key (Razorpay/Stripe dedup)

Result: user charged exactly once, regardless of retries at ANY layer
```

---

## Common Mistakes

1. **POST endpoints without idempotency keys.** Every state-changing POST should accept an idempotency key. No exceptions for financial operations.

2. **Check-then-insert in separate transactions.** Race condition: two concurrent requests both pass the "not processed" check. Use `INSERT ON CONFLICT` (atomic) instead.

3. **Using `balance += 100` instead of `SET balance = 500`.** Relative operations never idempotent. Absolute operations always are. Redesign as absolute.

4. **Not cleaning up old keys.** Idempotency store grows forever. Keys older than 24-48 hours can be deleted.

5. **Assuming Kafka `enable.idempotence=true` solves everything.** Only prevents duplicate writes TO Kafka. Doesn't make consumer idempotent — that's your code.

---

## Quiz + Answers

**Q1.** `POST /wallet/add-money` with `{amount: 500}`. Timeout, client retries. Wallet has ₹1000 instead of ₹500. How to fix?

**A1.** (1) Require `Idempotency-Key` header. Same key → return cached result, no double-add. (2) Redesign DB: `INSERT INTO transactions (txn_id, amount) VALUES (key, 500) ON CONFLICT DO NOTHING`. Balance = `SUM(amount)`. INSERT is idempotent (same txn_id → DO NOTHING).

---

**Q2.** Kafka consumer processes `PaymentCharged` events, sends confirmation emails. Customer got 3 identical emails. What went wrong?

**A2.** Consumer crashed, restarted, re-read same event (D37). Fix: check `event_id` in `processed_events` table. If exists → skip. If not → send email AND insert event_id in ONE transaction. Or pass event_id as idempotency key to email provider.

---

**Q3.** Two concurrent requests with same `Idempotency-Key: "idk_xyz"`. Both check DB — key not found. Both proceed. Double charge. How to prevent?

**A3.** Use `INSERT ON CONFLICT DO NOTHING` — single atomic SQL statement. Request A inserts first. Request B's insert hits unique constraint → fails → reads A's stored result. The dedup check and business logic must be in the SAME atomic transaction, not separate steps.

---

## Follow-up: Idempotent Consumer in JavaScript

```js
async function processEvent(event, consumer) {
  const eventId = event.eventId;

  // Step 1: Already processed?
  const already = await db.query(
    "SELECT 1 FROM processed_events WHERE event_id = $1", [eventId]
  );
  if (already.rows.length > 0) return; // skip duplicate

  // Step 2: Process + record in ONE transaction
  const client = await db.getClient();
  try {
    await client.query("BEGIN");         // pencil mode — nothing permanent yet
    await client.query(
      "INSERT INTO payments (payment_id, amount) VALUES ($1, $2)",
      [event.data.orderId, event.data.amount]
    );
    await client.query(
      "INSERT INTO processed_events (event_id) VALUES ($1)", [eventId]
    );
    await client.query("COMMIT");        // PEN — both saved permanently
  } catch (err) {
    await client.query("ROLLBACK");      // erase everything since BEGIN
    throw err;
  } finally {
    client.release();
  }

  // Step 3: Tell Kafka "I'm done"
  await consumer.commitOffset();
  // If crash here → Kafka resends → Step 1 catches duplicate ✅
}
```

**BEGIN/COMMIT explained:** `BEGIN` = changes in pencil (only you see). `COMMIT` = write in pen (permanent). `ROLLBACK` = erase pencil. If crash before COMMIT → auto-rollback → neither INSERT exists → safe to retry.

---

## Follow-up: Which Operations Are DB vs Kafka

| Step | Who | Talks to | Operation |
|---|---|---|---|
| Save order | Order Service | **Order DB** | `INSERT INTO orders` |
| Publish event | Order Service | **Kafka** | `producer.send()` |
| Pull event | Payment Service | **Kafka** | `consumer.poll()` |
| Check duplicate | Payment Service | **Payment DB** | `SELECT FROM processed_events` |
| Process + mark | Payment Service | **Payment DB** | `INSERT payments + INSERT processed_events` |
| Commit offset | Payment Service | **Kafka** | `consumer.commitOffset()` |

Each service has its OWN database. `processed_events` is in Payment Service's DB — NOT Order Service's.

---

## Follow-up: Re-reading vs Re-publishing

Kafka never re-publishes. Producer publishes ONCE. If consumer crashes before offset commit, the consumer re-READS the same single copy.

```
Producer publishes evt_abc123 → ONE copy in Kafka → done forever

Consumer reads it → processes → crashes before commit
Consumer restarts → reads SAME copy again (Kafka didn't duplicate it)

Idempotency protects against re-READING, not re-PUBLISHING.
```

---

## Follow-up: Idempotency at BOTH Layers

Need protection at Order Service (prevent duplicate orders) AND Payment Service (prevent duplicate payments):

```
Layer 1 — Order Service:
  Client sends Idempotency-Key header → Order Service checks "already created?"
  → prevents duplicate orders AND duplicate events being published

Layer 2 — Payment Service consumer:
  Checks event_id in processed_events → prevents duplicate processing
  → needed because consumer can re-read after crash (even if only one event exists)
```

| Layer | Protects against | How |
|---|---|---|
| Order Service | User clicking twice, browser retry | `idempotency_key` in orders table |
| Kafka producer | Producer retry publishing same event | `enable.idempotence=true` |
| Payment consumer | Consumer re-reading after crash | `event_id` in processed_events |
