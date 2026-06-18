# D44 — Event-Driven Architecture: Pub/Sub, Event Sourcing

---

## Why It Matters

In microservices (D42), services communicate via sync (HTTP/gRPC) or async (Kafka). EDA formalizes async into an architectural style: **services announce what happened, not what to do.**

> **Real case: Amazon — command-driven → event-driven**
> Order Service had 15 direct API calls to downstream services. Adding fraud detection = modifying Order Service. Fix: Order Service publishes `OrderPlaced`. Fraud, Shipping, Email subscribe independently. Adding a new consumer = zero changes to Order Service. Enabled team independence.

> **Real case: Uber — event sourcing for trips**
> Trips stored as single row, updated at each step. Disputes had no history. Fix: every state change stored as immutable event: `TripRequested`, `DriverMatched`, `TripStarted`, `TripCompleted`. Full audit trail. Billions of events/day.

---

## Commands vs Events [CRITICAL]

```
COMMAND:  "Hey Payment Service, charge $99"      ← sender knows receiver, waits
EVENT:    "OrderPlaced {order_id: 456, amount: $99}"  ← sender doesn't know who listens
```

| Dimension | Command | Event |
|---|---|---|
| **Direction** | "Do this" | "This happened" |
| **Coupling** | Sender knows receiver | Sender doesn't know receivers |
| **Adding consumers** | Change sender code | Zero changes to sender |
| **Response** | Waits for response | Fire and forget |

**One-line answer:** Commands are instructions. Events are announcements. Events decouple sender from all receivers.

---

## Pub/Sub Pattern [CRITICAL]

Publishers emit events. Subscribers listen. Broker (Kafka) sits in the middle.

```
Order Service ──publish──→ Kafka "order-events" ──→ Payment Service
User Service  ──publish──→                       ──→ Inventory Service
                                                 ──→ Email Service
                                                 ──→ Fraud Service (added later, zero changes)
```

### Event Design [IMPORTANT]

```json
{
  "event_id": "evt_abc123",
  "event_type": "OrderPlaced",
  "timestamp": "2026-06-18T21:30:00Z",
  "source": "order-service",
  "data": {
    "order_id": "ord_456",
    "user_id": "usr_789",
    "items": [{"product": "laptop", "qty": 1}],
    "total_amount": 99900,
    "currency": "INR"
  }
}
```

| Approach | Payload | Pros | Cons |
|---|---|---|---|
| **Thin event** | Just IDs: `{order_id: 456}` | Small | Consumer calls back for details (coupling!) |
| **Fat event** | All data included | Fully decoupled, no callback | Larger messages |

**Recommendation:** Fat events. Decoupling is the whole point.

---

## Event Sourcing [CRITICAL]

Traditional: store **current state** (UPDATE row). Event sourcing: store **every event** (append-only), derive state by replaying.

### Traditional
```
Orders Table: order_id=456, status="shipped", amount=₹999
History: GONE. Only latest UPDATE survived.
```

### Event Sourcing
```
Event Store (append-only):
  1. OrderCreated  {user: Aryan, items: laptop}   10:00 AM
  2. PaymentRecvd  {amount: ₹999, method: UPI}    10:01 AM
  3. OrderConfirmed {}                              10:01 AM
  4. ItemShipped    {tracking: AWB123}              2:00 PM

Current state: replay all 4 → status = "shipped"
Full history preserved.
```

### Use Cases

| Use Case | How event sourcing helps |
|---|---|
| **Audit trail** | Every change recorded. Required for banking, healthcare. |
| **Debugging** | Replay events to see exactly which event caused bad state. |
| **Rebuild views** | Fix bug, replay all events → correct data. Like Kafka offset reset (D37). |
| **Time travel** | Replay up to specific time → exact past state. |
| **Undo** | Append compensating event (`OrderCancelled`). Never delete/update. |

### Event Sourcing + Kafka = Natural Fit

```
Kafka:          [offset 0][offset 1][offset 2]...  ← append-only, replayable
Event store:    [seq 1][seq 2][seq 3]...            ← append-only, replayable
```

Kafka can BE your event store.

---

## CQRS [IMPORTANT]

**CQRS** = Command Query Responsibility Segregation. Separate write model (event store) from read model (materialized view).

```
WRITE SIDE                              READ SIDE
──────────                              ─────────
PlaceOrder command                      GetOrder query
      │                                       │
      ▼                                       ▼
Event Store ──events──→ Projection ──→ Read Database
(append-only)           (async)        (SQL/Redis/ES)
                                       order_id: 456
                                       status: "shipped"
```

- Writes: fast (just append)
- Reads: fast (pre-computed, optimized)
- Trade-off: **eventual consistency** — read side slightly behind write side

---

## Choreography vs Orchestration [CRITICAL]

### Choreography — No Central Controller
```
OrderPlaced → Payment reacts → publishes PaymentCompleted
            → Inventory reacts → publishes StockReserved
            → Notification reacts to PaymentCompleted → sends email
```
Services react independently. Like dancers following music.

### Orchestration — Central Controller
```
Orchestrator:
  Step 1: call Payment → "charge card" → done
  Step 2: call Inventory → "reserve stock" → done
  Step 3: call Notification → "send email" → done
```
One conductor directing each service.

| Dimension | Choreography | Orchestration |
|---|---|---|
| Control | No central controller | One orchestrator |
| Coupling | Very loose | Orchestrator knows all services |
| Visibility | Hard to trace full flow | Easy — flow in one place |
| Adding steps | New subscriber, zero changes | Modify orchestrator |
| Debugging | Hard | Easier |
| Best for | Simple, independent reactions | Complex flows with conditions/retries |

**Most systems use both.** Choreography for simple reactions (analytics, notifications). Orchestration for complex flows (order fulfillment → Saga pattern D47).

---

## When NOT to Use Event-Driven [CRITICAL]

| Problem | Why EDA makes it worse |
|---|---|
| Simple CRUD app | Events add complexity for zero benefit |
| Need immediate consistency | Events = eventual consistency |
| Small team (< 5 engineers) | Debugging event flows is hard |
| Ordering across entities | Cross-partition events can arrive out of order (D36) |

**Common mistakes:**

1. **Events when function call works.** In monolith, `paymentModule.charge()` is fine. Kafka event → consumer → charge = 10x complexity for zero benefit.

2. **No dead letter queue (DLQ).** Failed events need somewhere to go. Without DLQ, they're lost or block the partition.

3. **No schema evolution.** Event schemas change over time. Need backward-compatible changes — same as protobuf field numbers (D43).

---

## Quiz + Answers

**Q1.** Order Service calls Payment, Inventory, Email, Analytics (all sync). Adding Fraud Detection = what change? How does EDA change this?

**A1.** Current: modify Order Service, add 5th HTTP call. Order depends on 5 services. EDA: Order publishes `OrderPlaced`. All 5 subscribe independently. Adding Fraud = new consumer subscribing to existing topic. Zero changes to Order Service.

---

**Q2.** Banking app stores balances in `accounts` table. Customer disputes ₹50,000 transfer from 3 weeks ago. Can you prove what happened? How does event sourcing help?

**A2.** Current: only current balance stored. No record of when/who/what. Event sourcing: every change is an event — `Deposited`, `Transferred{to, amount, initiated_by}`. Replay to see exact state before/after. Required for banking compliance.

---

**Q3.** Food delivery app, 4 engineers. Senior dev proposes event sourcing for everything (registration, menus, orders, delivery). Your response?

**A3.** Push back on "everything." Event sourcing justified for orders/delivery (audit trail, dispute resolution, debug stuck orders). For registration and menus → simple CRUD, event sourcing is overkill. 4 engineers can't afford debugging complexity everywhere. Apply surgically where audit/replay benefits justify cost.

---

## Follow-up: Commands Use REST or gRPC

The synchronous command pattern (Order Service → "charge this card" → Payment Service) uses REST or gRPC — either works:

```
REST:  POST http://payment-service:8080/charge  body: {card, amount}
gRPC:  paymentService.chargeCard(card, amount)
```

Transport doesn't change the architectural point — either way, sender KNOWS receiver and WAITS. That's what makes it a command (coupled), not an event (decoupled).

---

## Follow-up: Where Events Are Stored — DB vs Kafka vs Both

| Option | Storage | Pros | Cons |
|---|---|---|---|
| **Database only** | PostgreSQL `order_events` table | ACID, queryable (`WHERE order_id=456`), forever | Low throughput, other services can't subscribe |
| **Kafka only** | Kafka topic | High throughput, built-in pub/sub | No query language, retention limit (default 7 days) |
| **Both (production standard)** | DB = permanent truth, Kafka = distribution | Queryable + subscribable + permanent | Need to keep them in sync |

### Transactional Outbox Pattern (keeping DB + Kafka in sync)

```
Step 1: ONE database transaction:
  INSERT INTO order_events (...)    ← source of truth
  INSERT INTO outbox (...)          ← copy, waiting to be published
  COMMIT                            ← atomic

Step 2: Background worker:
  SELECT FROM outbox WHERE published = false
  → kafka.publish(event)
  → UPDATE outbox SET published = true
  
  If Kafka fails → retry next poll. If worker crashes → restarts, picks up unpublished rows.
```

---

## Follow-up: EDA in Monoliths vs Microservices

| EDA concept | Monolith? | Microservices? |
|---|---|---|
| **Pub/Sub** (service-to-service events) | ❌ No services to decouple — function calls are fine | ✅ Primary use case |
| **Async offloading** (Kafka for background work) | ✅ Yes — email, PDF, slow tasks | ✅ Yes |
| **Event sourcing** (append-only storage) | ✅ It's a DB pattern, works anywhere | ✅ Yes |
| **CQRS** (separate read/write models) | ✅ It's a DB pattern, works anywhere | ✅ Yes |

Pub/Sub between modules inside a monolith = pointless (adding network hops between function calls). Event sourcing and CQRS are data patterns that work anywhere.
