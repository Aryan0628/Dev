# D35 — Message Queues Overview: Why Async Matters, At-Least-Once vs Exactly-Once

---

## Why It Matters — Cascade Failure from Synchronous Coupling

Without a queue, services call each other directly and wait for a response. If the downstream service is slow or down, the caller hangs or fails too. One slow service cascades across everything connected to it.

> **Real case: Amazon's product pages — early 2000s**
> What happened: Pages called recommendations, inventory, pricing, reviews synchronously. When recommendations slowed, every page load slowed with it.
> Root cause: Synchronous coupling — page renderer waited for every downstream service.
> Cost / impact: Amazon has stated 100ms of latency = ~1% revenue loss. Recommendations alone cost millions per year.
> Fix: Moved non-critical services (recommendations, reviews) to async messaging. Page renders when product data is ready; recommendations load asynchronously. This was the forcing function behind Amazon's service-oriented architecture mandate.

---

## Core Concept

A message queue is a **post office between services**. The producer drops a message and walks away immediately. The broker stores it. The consumer picks it up when ready. The two services never need to be up at the same time or run at the same speed.

```
Without queue:
Checkout → (waits) → Email Service → (waits) → done
           ↑ Email Service down? Checkout fails.

With queue:
Checkout → drops message → done (immediately)
                            ↓
             Email Service → pulls when ready → processes
             ↑ Email Service down? Message waits safely in queue.
```

### Four Moving Parts

| Component | Role |
|---|---|
| **Producer** | Service that sends the message |
| **Broker** | Server that stores and routes messages |
| **Queue/Topic** | Named channel the messages flow through |
| **Consumer** | Service that reads and processes messages |

---

## How It Works Internally

### Message Lifecycle

```
Producer → send("charge: $50") → Broker (writes to disk) → deliver → Consumer
                                                                          ↓ processes
                                  Broker deletes message ← ACK ← Consumer
```

**ACK (Acknowledgment):** Consumer signals to broker: "Received and processed successfully. You can delete this." The broker holds the message on disk until the ACK arrives. This is what makes the system durable.

### Push vs Pull

| Model | Who drives delivery | Risk | Used by |
|---|---|---|---|
| **Push** | Broker sends as fast as it can | Consumer gets overwhelmed | RabbitMQ (default) |
| **Pull** | Consumer polls when ready | Slight polling latency | SQS, Kafka |

**Pull wins in production.** Consumer controls its own rate. If DB is under load, it pulls less. This is **backpressure** — slow downstream naturally slows the pipeline without crashing it.

### Visibility Timeout

When a broker delivers a message, it hides it from other consumers for a fixed window (e.g., 30 seconds). If no ACK arrives before timeout, the broker assumes the consumer crashed and redelivers. This is how SQS implements at-least-once.

---

## Delivery Semantics [CRITICAL]

| Semantic | Message loss? | Duplicates? | Common in |
|---|---|---|---|
| **At-most-once** | Yes, possible | Never | Metrics, logs where loss is OK |
| **At-least-once** | Never | Yes, possible | Most production systems |
| **Exactly-once** | Never | Never | Financial transactions (complex) |

**One-line answer:** At-least-once is what you usually get. Exactly-once is what you usually want. The gap is bridged by idempotency.

---

## Why At-Least-Once Creates Duplicates — The ACK Timing Problem [CRITICAL]

```
1. Broker delivers: "charge user $50, payment_id=abc123"
2. Consumer processes — money deducted ✓
3. Consumer sends ACK
4. Network drops the ACK — broker never receives it
5. Visibility timeout expires — broker assumes consumer crashed
6. Broker redelivers: "charge user $50, payment_id=abc123"
7. Consumer charges the user AGAIN ✗
```

You cannot atomically "process a message AND send an ACK" across a distributed system. Processing happens in your app. ACK travels over a network. A crash or network drop between steps 2 and 3 causes redelivery. This is physics, not a bug.

---

## The Fix: Idempotent Consumers [CRITICAL]

**Idempotent:** An operation that produces the same result whether run once or ten times.

**Non-idempotent (dangerous):**
```python
def process_payment(message):
    charge_user(message["user_id"], message["amount"])  # Runs twice? Charges twice.
    send_ack()
```

**Idempotent (safe):**
```python
def process_payment(message):
    payment_id = message["payment_id"]

    if already_processed(payment_id):   # check DB for this ID
        send_ack()                      # already done, just ack
        return

    charge_user(message["user_id"], message["amount"])
    mark_as_processed(payment_id)       # record it
    send_ack()
```

Producer assigns a unique ID to every message. Consumer checks if that ID was already processed. Second delivery → already processed → skip.

> **Real case: Robinhood 2021 — duplicate trade executions**
> What happened: A retry mechanism resubmitted trade orders during high latency. Without idempotency checks, some users had orders executed twice — money deducted twice, stocks purchased twice.
> Root cause: At-least-once delivery + non-idempotent consumer. Order creation didn't check if an order with that request ID already existed.
> Cost / impact: Manual cancellation of duplicate orders, refunds, regulatory disclosures, trust damage.
> Fix: Each order request carries a unique idempotency key. Before creating an order, check: "has this key been processed?" If yes, return the existing order. If no, create it.

---

## Why Exactly-Once Is Hard [IMPORTANT]

True exactly-once requires the broker and the consumer's database to participate in the same transaction: "deliver the message AND commit my DB write" as one atomic operation. That spans two different systems — a distributed transaction (covered in D46/D47).

Kafka achieves exactly-once through:
- **Idempotent producers:** Sequence numbers prevent broker-side duplicates
- **Transactional consumers:** Offset commits happen in the same transaction as the DB write

But: expensive to configure, throughput cost, requires careful setup.

**In practice:** Most production systems use **at-least-once + idempotent consumers**. This achieves the effective guarantee of exactly-once without distributed transaction complexity.

**One-line answer:** Exactly-once = at-least-once delivery + idempotency in the consumer.

---

## Dead Letter Queue (DLQ) [IMPORTANT]

**DLQ (Dead Letter Queue):** Where messages go after failing processing N times (typically 3-5).

Without a DLQ: a single malformed message retries forever, blocks the queue, starves all other messages. This is a **poison message**.

```
Normal queue: msg1, msg2, [bad msg3], msg4, msg5
                              ↓ fails 3x
                         DLQ: [bad msg3]
Normal queue continues: msg1, msg2, msg4, msg5 ✓
```

Inspect the DLQ to debug why messages failed. Fix the consumer bug. Replay from DLQ.

---

## Message Persistence + Connection to D26 [IMPORTANT]

Dedicated brokers (SQS, RabbitMQ, Kafka) write messages to disk before ACKing the producer. "Message sent" = "stored durably."

Redis lists (`LPUSH` / `RPOP`) can work as a simple queue. But if Redis runs without AOF persistence, all messages vanish on crash. Common trap: Redis works fine in dev, used as a queue in prod, someone disables persistence for speed — unprocessed jobs disappear on restart.

---

## Common Queue Systems [IMPORTANT]

| System | Delivery model | Ordering | Exactly-once | Best fit |
|---|---|---|---|---|
| **SQS Standard** | Pull | No guarantee | No | Simple async tasks, high throughput |
| **SQS FIFO** | Pull | Yes, strict | Yes | Order-sensitive tasks |
| **RabbitMQ** | Push (default) | Per-queue FIFO | With transactions | Complex routing, per-message TTL |
| **Kafka** | Pull | Per-partition | Yes (complex) | High-throughput event streams, replay |
| **Redis Lists** | Pull | FIFO | No | Simple, low-volume background jobs |

**Key Kafka distinction:** Kafka is not just a queue — it's a log. Messages are retained even after consumption. Consumers can replay from any point in history. D36/D37 covers Kafka in depth.

---

## When NOT to Use a Queue

- **When you need a synchronous response** — if the user is waiting for a result (e.g., coupon validation), async adds latency with no benefit. Use direct API call.
- **When strict global ordering + high throughput** — expensive combination. Design around needing this if possible.
- **Simple, low-traffic tasks** — a cron job is often simpler and more debuggable than a queue + consumer deployment.

---

## Most Common Mistakes in Production

1. **Non-idempotent consumers with at-least-once delivery.** Assuming "queue delivers once" and skipping idempotency. The first network blip causes duplicate processing. For money, DB writes, external API calls — always assume the message can arrive twice.

2. **No DLQ configured.** Bad message retries forever. Queue backs up. All other messages stuck. Consumers spin at 100% CPU retrying the same failure. Always configure a DLQ with a max retry count.

---

## Queue Depth vs Consumer Lag

**Queue depth** = number of messages waiting. Growing depth = consumers slower than producers.

But queue depth is a **lagging indicator** — by the time an alert fires, you're already behind. Better metric: **consumer lag** = how old is the oldest unconsumed message.

---

## Quiz + Answers

**Q1.** Your payment service deducts money from a user's account. Before it can ACK, the consumer crashes. The broker redelivers the message. What happens, and what's the fix?

**A1.** User gets charged twice. Fix: idempotent consumer. Message carries a unique `payment_id`. Before charging, check if this `payment_id` is already in the `processed_payments` table. If yes, skip the charge and ACK. If no, charge, record, ACK.

---

**Q2.** What's the difference between at-least-once and exactly-once delivery? Why can't you just always use exactly-once?

**A2.** At-least-once: delivered 1+ times, possible duplicates. Exactly-once: delivered exactly once, no loss, no duplicates. Exactly-once is hard because ACKing and processing can't happen atomically across two systems — a crash between processing and ACKing causes redelivery. True exactly-once requires a distributed transaction across broker + consumer storage, which is expensive and complex. In practice: at-least-once + idempotent consumer achieves the same result at lower cost.

---

**Q3.** Queue depth is growing: 10,000 messages waiting and climbing. What are two possible causes and two possible responses?

**A3.** Causes: (1) Consumer is slow — processing takes longer than before (slow DB query, external API timeout). (2) Producer spike — more messages being published than usual (traffic burst). Responses: (1) Scale out — add more consumer instances (requires consumers to be stateless and idempotent). (2) Investigate and optimize the consumer — find what slowed processing. Accepting temporary lag is a third option only if the spike is clearly short-lived.
