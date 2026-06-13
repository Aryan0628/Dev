# D37 — Kafka Internals Part 2: Consumer Groups, Retention, Rebalancing

---

## Why It Matters — The Other Half of Kafka

D36 covered how data gets **into** Kafka (producers, partitions, offsets, storage tricks). This session covers how data gets **out** reliably — consumer groups, offset management, retention, and rebalancing. These are the mechanisms that make Kafka production-grade.

> **Real case: Slack — 2020, thundering herd during COVID surge**
> What happened: Multiple Kafka consumer instances crashed simultaneously during traffic spike. Each restart triggered Kafka's eager rebalancing — ALL consumers stopped processing during each rebalance. Cascade: crash → rebalance → slow recovery → timeout → crash → rebalance.
> Root cause: Eager rebalancing is "stop-the-world" — pauses every consumer in the group to reassign partitions.
> Cost / impact: Degraded messaging for millions of users during peak remote-work traffic.
> Prevention: Use **cooperative/incremental rebalancing** (Kafka 2.4+) where only affected partitions pause.

---

## Consumer Groups [CRITICAL]

### What They Are

A **consumer group** = a named set of consumer instances that **cooperate** to read from a topic.

**Rules:**
1. Each partition → assigned to exactly ONE consumer within the group
2. Each consumer can own multiple partitions
3. Every message is processed by exactly one consumer in the group

```
Topic "orders" (4 partitions)
Consumer Group: "billing-service"

   P0 ──→ Consumer A     (assigned P0 and P1)
   P1 ──→ Consumer A
   P2 ──→ Consumer B     (assigned P2)
   P3 ──→ Consumer C     (assigned P3)
```

### Multiple Consumer Groups = Independent Consumption [CRITICAL]

Different groups read the same data independently. Each group tracks its own offsets. Producer writes once, N groups read everything.

```
Consumer Group: "billing-service"
   P0 ──→ Billing-1 (offset 500)
   P1 ──→ Billing-2 (offset 350)

Consumer Group: "analytics-service"
   P0 ──→ Analytics-1 (offset 200)    ← reading slower, fine
   P1 ──→ Analytics-1 (offset 200)

Consumer Group: "fraud-detection"
   P0 ──→ Fraud-1 (offset 500)
   P1 ──→ Fraud-2 (offset 500)
```

Adding a new consumer group = zero changes to producers or other consumers. Deploy and start reading.

| D35 Traditional Queue | Kafka Consumer Group |
|---|---|
| One message → one consumer (competing) | One message → one consumer **per group** |
| Adding a new consumer splits the work | Adding a new **group** gets ALL messages independently |

**One-line answer:** A consumer group is a team that splits work internally, but different teams each get their own complete copy.

---

## Offset Management [CRITICAL]

### How Consumers Track Progress

Consumers **commit offsets** to a special internal topic called `__consumer_offsets`. This tells Kafka "I've processed up to here."

### The Commit Timing Problem — Delivery Guarantees Made Concrete

| Strategy | How | Risk | Guarantee | Use when |
|---|---|---|---|---|
| **Auto-commit** | Kafka commits every 5s automatically | Crash between process & auto-commit → duplicates | At-least-once | Prototyping |
| **Commit-after-process** (manual) | You commit after processing | Crash between process & commit → last message duplicated | At-least-once (controlled) | **Most production systems** |
| **Commit-before-process** | You commit before processing | Crash after commit, before process → messages LOST | At-most-once | Metrics, sampling |
| **Exactly-once (transactional)** | Kafka transactions | Complex, higher latency | Exactly-once (within Kafka only) | Kafka-to-Kafka pipelines |

### Auto-commit failure scenario:

```
t=0s  poll() → get messages [3, 4, 5]
t=1s  process message 3 ✓
t=2s  process message 4 ✓
t=3s  💥 crash (auto-commit never fired — only happens every 5s)
      Committed offset still 3.
      Restart → reads from 3 → messages 3, 4 processed AGAIN.
```

### Commit-after-process failure scenario:

```
poll() → get message [4]
process message 4 ✓
💥 crash before commitSync()
Committed offset still 4.
Restart → reads from 4 → message 4 processed AGAIN.
Result: only last message duplicated, not whole batch.
```

### Exactly-Once Semantics (EOS) [IMPORTANT]

Works **only for Kafka-to-Kafka pipelines** — producer output and consumer offset commit in one atomic Kafka transaction.

```
Kafka Topic A ──→ Consumer (transform) ──→ Kafka Topic B
                       └── offset commit + output write = ONE transaction
```

**Does NOT extend to external systems.** Kafka + PostgreSQL = two systems, no cross-system atomic commit. Use at-least-once + idempotent writes (upsert with unique message ID).

**⚠️ `enable.idempotence=true` on producer only prevents duplicate writes TO Kafka on retries. It says nothing about consumer-side duplicates.**

---

## Retention — How Long Kafka Keeps Data [CRITICAL]

### Time-Based (default)

```
log.retention.hours = 168  (7 days)
```

Messages older than 168 hours → oldest log segments deleted.

### Size-Based

```
log.retention.bytes = 10737418240  (10 GB per partition)
```

When partition exceeds limit → oldest segments deleted. Combines with time — whichever triggers first wins.

### Compacted Topics [IMPORTANT]

Keeps only the **latest message per key**. Older values for same key are deleted.

```
Before compaction:  [alice=Alice] → [bob=Bob] → [alice=Alicia] → [bob=Robert]
After compaction:   [alice=Alicia] → [bob=Robert]
```

| Retention type | Deletes | Use case |
|---|---|---|
| Time-based | Messages older than X hours | Event streams (orders, logs) |
| Size-based | Oldest when partition > X bytes | Bounded storage |
| Compaction | Older values for same key | State topics (profiles, config) |

> **Real case: Uber — consumer lag exceeded retention**
> What happened: Analytics consumer disabled for maintenance, forgotten for 8 days. Retention was 7 days. Restarted consumer's committed offset pointed to deleted messages.
> Cost / impact: 1 day of analytics data permanently lost.
> Prevention: Monitor consumer lag. Alert when lag exceeds 50% of retention window.

**⚠️ Critical: If consumer lag exceeds retention, unconsumed messages are permanently deleted.**

---

## Rebalancing — What Happens When Consumers Change [CRITICAL]

### Triggers

1. New consumer joins (deploy, autoscale)
2. Consumer leaves (crash, shutdown)
3. Consumer stops heartbeating (GC pause, network)
4. Partition count changes

### Group Coordinator

One specific broker manages membership for each consumer group. Consumers send heartbeats every `heartbeat.interval.ms` (default 3s). Missing heartbeats for `session.timeout.ms` (default 45s) → declared dead → rebalance.

### Eager Rebalancing (Legacy, Pre-2.4) [IMPORTANT]

**Stop-the-world.** ALL consumers stop, all partitions revoked, reassigned from scratch, all resume.

```
Consumer C joins → ALL consumers pause → reassign everything → resume
Total pause: seconds to minutes for large groups.
```

### Cooperative (Incremental) Rebalancing (Kafka 2.4+) [CRITICAL]

**Only affected partitions move.** Other consumers keep processing.

```
Consumer C joins → only partitions that move are paused → rest keep running
Total disruption: minimal — just the moved partitions.
```

```java
// Enable cooperative rebalancing
props.put("partition.assignment.strategy", 
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

### Assignment Strategies [IMPORTANT]

| Strategy | Behavior | Best for |
|---|---|---|
| **Range** (default) | Contiguous partition ranges per topic | Simple setups |
| **RoundRobin** | Distribute one-by-one | Even distribution |
| **StickyAssignor** | RoundRobin + minimizes movement | Fewer reassignments |
| **CooperativeStickyAssignor** | Sticky + cooperative (no stop-the-world) | **Production** |

### Key Configs [CRITICAL]

| Config | Default | What it does | Recommendation |
|---|---|---|---|
| `session.timeout.ms` | 45s | Time before missing consumer is declared dead | 45s good. Too low = false positives during GC |
| `heartbeat.interval.ms` | 3s | Heartbeat frequency | 1/3 of session.timeout |
| `max.poll.interval.ms` | 300s (5 min) | Max time between poll() calls before eviction | Must be > longest message processing time |
| `max.poll.records` | 500 | Max messages per poll() | Tune to processing speed |

### The Sneaky One: `max.poll.interval.ms`

Heartbeats are sent by a **background thread** — they keep flowing even if processing is stuck. But if `poll()` isn't called within `max.poll.interval.ms`, consumer is evicted.

```
poll() → 500 records → processing takes 6 minutes
max.poll.interval.ms = 300000 (5 min)
At 5 min: coordinator evicts consumer → rebalance
Consumer finishes, tries to commit → FAILS (no longer owns partitions)
```

**Fix:** Increase `max.poll.interval.ms` or decrease `max.poll.records`.

---

## Static Group Membership [IMPORTANT]

Assign persistent `group.instance.id` to each consumer. On restart with same ID → Kafka recognizes it → no rebalance if it returns within `session.timeout.ms`.

```java
props.put("group.instance.id", "billing-consumer-1");
```

```
Without static membership:  restart → new member ID → rebalance
With static membership:     restart → same ID → no rebalance if within timeout
```

**When to use:** Kubernetes pods that restart during deploys.

---

## The Consumer Poll Loop — Canonical Pattern [CRITICAL]

```java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("group.id", "billing-service");
props.put("key.deserializer", "...StringDeserializer");
props.put("value.deserializer", "...StringDeserializer");
props.put("enable.auto.commit", "false");  // manual commit
props.put("partition.assignment.strategy", 
    "...CooperativeStickyAssignor");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("orders"));

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, String> record : records) {
            processBillingEvent(record);  // your business logic
        }
        consumer.commitSync();  // commit after processing batch
    }
} finally {
    consumer.close();  // graceful leave → immediate rebalance (no 45s wait)
}
```

**`consumer.close()` sends LeaveGroup → immediate rebalance.** Always close gracefully.

---

## Consumer Lag — The #1 Kafka Metric [CRITICAL]

**Consumer lag** = latest produced offset − committed consumer offset.

```bash
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
    --describe --group billing-service

# GROUP           TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# billing-service orders  0          7500            10000           2500
# billing-service orders  1          4200            4200            0
```

| Lag | Meaning | Action |
|---|---|---|
| 0 | Keeping up perfectly | None |
| Growing steadily | Consumer slower than producer | Add consumers (up to partition count) |
| Spiking | Stuck or rebalancing | Check processing, rebalance storms |
| Approaching retention | **Data loss risk** | 🚨 Alert. Fix consumer or increase retention |

**Production setup:** Export lag to Prometheus/Grafana. Alert at 50% of retention time.

---

## Full Read Path — End to End

```
Consumer App                                    Kafka Cluster
     │                                               │
     │  1. new KafkaConsumer(group="billing")        │
     │  2. subscribe("orders")                       │
     │                                               │
     │ ──── FindCoordinator ──────────────────────► │
     │ ◄─── Coordinator = Broker 2 ─────────────── │
     │                                               │
     │ ──── JoinGroup ────────────────────────────► │ Broker 2 (Coordinator)
     │ ◄─── Assignment: you get P0, P1 ─────────── │
     │                                               │
     │  poll() ──── Fetch P0 from offset 0 ────────► │ Broker X (P0 leader)
     │  poll() ──── Fetch P1 from offset 0 ────────► │ Broker Y (P1 leader)
     │ ◄─── records ──────────────────────────────── │
     │                                               │
     │  process records                              │
     │  commitSync() ── offset 5 (P0) ────────────► │ Broker 2 (__consumer_offsets)
     │              ── offset 3 (P1) ────────────►   │
     │                                               │
     │  Background: heartbeat every 3s ────────────► │ Broker 2
```

---

## Common Mistakes Engineers Make

1. **Not tuning `max.poll.interval.ms`.** Default 5 min seems safe. 500 records × slow DB lookup = exceeded. Consumer evicted → rebalance → gets same partitions → processes again → evicted again → infinite loop.

2. **Deploying all consumers at once.** Every restart triggers rebalance. With eager rebalancing, entire group stops. **Always do rolling deploys** — restart one consumer at a time.

3. **No consumer lag monitoring.** Lag can silently grow until it exceeds retention → permanent data loss. Monitor lag, alert at 50% of retention.

4. **Using auto-commit for critical data.** Auto-commit timing is unpredictable. One crash → large batches reprocessed. Use manual commit-after-process.

5. **Thinking `enable.idempotence=true` = exactly-once.** It only deduplicates producer retries to Kafka. Consumer-side duplicates are a separate problem.

---

## Quiz + Answers

**Q1.** You have topic `orders` with 6 partitions, consumer group `billing-service` with 3 consumers. Consumer C crashes. Describe step-by-step what happens with cooperative sticky rebalancing.

**A1.**
1. Consumer C stops sending heartbeats.
2. After `session.timeout.ms` (45s), coordinator declares C dead.
3. Cooperative rebalance: only C's partitions (e.g., P4, P5) need reassignment. A and B **keep processing** P0–P3 without interruption.
4. Coordinator assigns P4 to A, P5 to B.
5. A and B start consuming P4, P5 from C's last committed offset.
6. If C processed but didn't commit → those messages processed again (at-least-once).
7. If C had called `consumer.close()` → rebalance would be immediate, no 45s wait.

---

**Q2.** Consumer group with `enable.auto.commit=true`, `auto.commit.interval.ms=5000`. Consumer polls 100 messages, processes 60, crashes at 61st. What happens on restart?

**A2.** Depends on timing of last auto-commit:
- If auto-commit fired after processing message 50: committed offset = 50. Restart reads from 50 → messages 50–60 processed **again**.
- If no auto-commit fired: committed offset = previous batch's end. All 60 messages reprocessed.
Either way: at-least-once semantics. Some messages processed twice.

---

**Q3.** Topic with `log.retention.hours=72` (3 days). Consumer group stops making progress for 4 days due to a bug. What happens when fixed and restarted?

**A3.** Committed offset points to deleted messages (past 3-day retention). Kafka returns `OffsetOutOfRangeException`. Behavior depends on `auto.offset.reset`:
- `latest` (default): jumps to latest. **All 4 days lost.**
- `earliest`: jumps to oldest available (3 days ago). **1 day lost**, 3 days recovered.
- `none`: throws exception, refuses to start. Engineer must manually decide — **safest for critical data.**

**Lesson:** Set `auto.offset.reset=none` for critical groups. Monitor lag. Alert at 50% of retention.
