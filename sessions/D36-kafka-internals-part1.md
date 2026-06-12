# D36 — Kafka Internals Part 1: Topics, Partitions, Offsets, Producers

---

## Why It Matters — The Log That Changed Backend Engineering

In D35, you learned that a message queue decouples producers from consumers. Drop a message, walk away, consumer picks it up later. But traditional queues (RabbitMQ, SQS) have a limitation: **once a message is consumed and ACKed, it's deleted**. The broker forgets it ever existed.

What if you need:
- **Multiple independent consumers** reading the same stream of events (analytics team, billing team, fraud team — each needs every order event)
- **Replay** — a bug in your billing consumer processed orders wrong for 3 days, you need to reprocess them all
- **Audit trail** — regulatory requirement to prove what happened and when

Traditional queues can't do this. Once deleted, the data is gone.

Kafka solves this by flipping the model: **it's not a queue, it's a distributed append-only log.** Messages are retained regardless of whether anyone consumed them. Consumers track their own position. Multiple consumers read the same data independently.

> **Real case: LinkedIn — 2011, the origin of Kafka**
> What happened: LinkedIn had dozens of services producing events (page views, profile updates, ad impressions) and dozens of systems consuming them (analytics, monitoring, data warehouse). Each producer-consumer pair had its own point-to-point pipeline. They had **O(N×M)** connections — every producer connected to every consumer directly.
> Root cause: No central event backbone. Adding one new consumer meant modifying every producer.
> Cost / impact: Engineering velocity collapsed. Adding a new analytics pipeline took weeks of integration work.
> Fix: Jay Kreps, Neha Narkhede, and Jun Rao built Kafka — a single distributed log that all producers write to and all consumers read from independently. O(N+M) connections instead of O(N×M). LinkedIn processes **7 trillion messages per day** through Kafka today.

---

## Core Concept — Kafka Is a Distributed Commit Log

Forget the word "queue" for now. Kafka is a **log** — an ordered, append-only sequence of records, stored on disk, that you can read from any position.

Think of a **library's checkout ledger** from the old days:
- Every book checkout is written as a new line at the bottom (append-only)
- The ledger is never erased — entries stay for the retention period
- Multiple librarians can read the ledger independently, each with their own bookmark
- One librarian reading entry #500 doesn't prevent another from reading entry #200

### Kafka vs Traditional Queue (from D35) [CRITICAL]

| Dimension | Traditional Queue (SQS, RabbitMQ) | Kafka |
|---|---|---|
| **Model** | Message queue — deliver and delete | Distributed log — append and retain |
| **After consumption** | Message deleted | Message stays (until retention expires) |
| **Multiple consumers** | Competing — only one gets each message | Independent — each reads everything |
| **Replay** | Impossible (data gone) | Read from any offset at any time |
| **Ordering** | Per-queue FIFO (best effort for SQS Standard) | **Per-partition** strict ordering |
| **Throughput** | Thousands/sec | **Millions/sec** |
| **Delivery** | Push (RabbitMQ) or Pull (SQS) | Pull always |

**One-line answer:** A traditional queue is a pipe — data flows through and disappears. Kafka is a tape recorder — data is written permanently and anyone can rewind to any point.

---

## How It Works Internally

### The Three Core Abstractions [CRITICAL]

```
                         Kafka Cluster
                    ┌─────────────────────┐
                    │   Topic: "orders"   │
                    │                     │
                    │  Partition 0: [0][1][2][3][4][5]  ← append here
                    │  Partition 1: [0][1][2][3]        ← append here
                    │  Partition 2: [0][1][2][3][4]     ← append here
                    │                     │
                    └─────────────────────┘
                      ↑                         ↑
                  Producer A               Consumer X (reading P0, offset 3)
                  Producer B               Consumer Y (reading P1, offset 1)
```

### 1. Topic [CRITICAL]

A **topic** is a named category of messages. It's the logical channel — like a queue name from D35.

Examples: `orders`, `user-signups`, `page-views`, `payment-events`

A topic is just a name. It contains no data itself. Data lives in **partitions**.

### 2. Partition [CRITICAL]

A **partition** is one physical log file on one broker's disk. It's where data actually lives.

Each topic is split into **N partitions** (you choose N at creation time — e.g., `orders` might have 6 partitions). Each partition is:

- **Append-only** — new messages are written to the end, never inserted in the middle
- **Ordered** — messages within ONE partition have a strict, guaranteed order
- **Immutable** — once written, a message is never modified
- **Stored on exactly one broker** (the partition leader — more on this below)

**Why multiple partitions?** Parallelism. In D35, you had one queue — one consumer processes messages one at a time. With Kafka:

- Topic with 1 partition → 1 consumer max (sequential, like a queue)
- Topic with 6 partitions → up to 6 consumers reading **in parallel**, each owning different partitions

Partitions are Kafka's unit of parallelism. More partitions = more consumers can work simultaneously = higher throughput.

**⚠️ Critical distinction: ordering is per-partition, NOT per-topic.**

```
Topic: "orders" (3 partitions)

Partition 0: [order-A] → [order-D] → [order-G]    ← A before D before G ✓
Partition 1: [order-B] → [order-E]                  ← B before E ✓
Partition 2: [order-C] → [order-F] → [order-H]    ← C before F before H ✓

Cross-partition? No guarantee that A came before B or C.
```

If you need ordering for a specific entity (e.g., all events for user_123 must be in order), you must ensure they all go to the **same partition**. This is done via the **partition key** — explained in the producer section below.

### 3. Offset [CRITICAL]

An **offset** is a sequential ID number assigned to each message within a partition. Starts at 0, increments by 1 for each new message.

```
Partition 0:  [offset 0] [offset 1] [offset 2] [offset 3] [offset 4]
                                                    ↑
                                        Consumer X is here (committed offset = 3)
                                        "I've processed 0, 1, 2. Next: read 3."
```

**This replaces ACK from D35.** In SQS/RabbitMQ, the consumer ACKs each message → broker deletes it. In Kafka, the consumer **commits its offset** → broker remembers where the consumer left off. The message stays.

| D35 Queue | Kafka |
|---|---|
| Consumer ACKs message #5 | Consumer commits offset 5 |
| Broker deletes message #5 | Broker keeps message #5 on disk |
| Other consumers can't see it | Other consumers can still read it |

If a consumer crashes at offset 3, it restarts and asks: "What was my last committed offset?" → 2. It resumes from offset 3. No data loss.

If a **new** consumer group starts, it can choose: start from offset 0 (replay everything) or from the latest offset (only new messages).

---

## Kafka's Storage Model — Why It's So Fast [CRITICAL]

This is the part most people skip, but it's why Kafka can handle **millions of messages per second** while writing to disk. It should be impossible — disk is slow, right? Three tricks:

### Trick 1: Sequential I/O (not Random I/O)

A traditional database writes data to random locations on disk — the disk head jumps around. This is slow (~100 writes/sec for spinning disk, ~10K for SSD).

Kafka only **appends to the end of a file**. The disk head never moves backwards. Sequential writes on a spinning disk: **600 MB/sec**. Faster than random writes on an SSD.

```
Traditional DB:     write to page 7, then 4002, then 51, then 8844  (random)
Kafka partition:    write, write, write, write →→→→→→→→→→→→→        (sequential)
```

**One-line answer:** Kafka is fast because it treats disk like a tape — only moves forward, never rewinds to write.

### Trick 2: OS Page Cache [IMPORTANT]

Kafka doesn't manage its own in-memory cache (unlike Redis from D26 which owns all its data in RAM). Instead, it lets the **OS page cache** handle caching.

When Kafka writes to a file, the OS keeps recently written data in RAM automatically. When a consumer reads, it often hits the page cache — never touches disk at all.

```
Producer writes → Kafka appends to file → OS keeps it in page cache (RAM)
Consumer reads  → OS serves from page cache → zero disk I/O
```

This means Kafka's JVM (Java Virtual Machine — the runtime Kafka runs on) heap stays small and stable. No GC (Garbage Collection — Java's automatic memory cleanup) pressure. The OS manages memory far more efficiently than any application-level cache.

### Trick 3: Zero-Copy Transfer [GOOD TO KNOW]

Normal path to send file data to a consumer over the network:

```
Disk → Kernel buffer → User-space buffer (Kafka process) → Kernel buffer → NIC (network card)
       4 copies, 2 context switches
```

Zero-copy path (Kafka uses `sendfile()` system call):

```
Disk → Kernel buffer → NIC
       1 copy, 0 user-space involvement
```

Kafka never even reads the message content into its own process memory when serving consumers. The OS transfers directly from page cache to the network card.

**One-line answer:** Kafka is fast because it does almost nothing — the OS does the heavy lifting via sequential I/O, page cache, and zero-copy.

---

## Brokers and the Cluster [CRITICAL]

A **broker** is one Kafka server process (one machine, or one container). A Kafka **cluster** is multiple brokers working together.

```
Kafka Cluster (3 brokers)
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Broker 0   │  │   Broker 1   │  │   Broker 2   │
│              │  │              │  │              │
│  orders-P0 ★ │  │  orders-P1 ★ │  │  orders-P2 ★ │
│  orders-P1 ○ │  │  orders-P2 ○ │  │  orders-P0 ○ │
│              │  │              │  │              │
└──────────────┘  └──────────────┘  └──────────────┘

★ = Leader replica (handles all reads/writes)
○ = Follower replica (passive copy for durability)
```

### Replication [CRITICAL]

Each partition has a **replication factor** (typically 3). One replica is the **leader** — all producer writes and consumer reads go to the leader. The other replicas are **followers** — they continuously copy data from the leader.

If a broker dies, one of the followers gets promoted to leader. No data loss (if configured correctly — see `acks` below).

**ISR (In-Sync Replicas):** The set of replicas that are fully caught up with the leader. If a follower falls behind (slow disk, network issue), it drops out of the ISR. Kafka only considers a write "safe" when it's replicated to all ISR members.

This connects directly to D08 (load balancing) — partitions are distributed across brokers for even load, and to D20 (database replication) — same leader/follower concept, same replication lag risks.

---

## The Producer — How Messages Get Into Kafka [CRITICAL]

### Producer Flow

```
Your App (Producer)
        │
        ▼
   1. Serialize message (key + value → bytes)
        │
        ▼
   2. Pick partition
      ├── Key provided? → hash(key) % num_partitions → deterministic partition
      └── No key?       → round-robin across partitions
        │
        ▼
   3. Add to internal batch buffer (per partition)
        │
        ▼
   4. Background sender thread flushes batch to partition leader broker
        │
        ▼
   5. Broker writes to leader log → replicates to followers → sends ACK
```

### Partition Key — How You Control Where Messages Land [CRITICAL]

The **key** is not the message content — it's a routing hint. Kafka hashes the key to decide which partition gets the message.

```python
# All events for user_123 go to the SAME partition → guaranteed ordering
producer.send(topic="orders", key="user_123", value={"item": "laptop", "price": 999})
producer.send(topic="orders", key="user_123", value={"item": "charger", "price": 29})
# Both → hash("user_123") % 3 = partition 1 (always the same)

# Events for user_456 might go to a DIFFERENT partition
producer.send(topic="orders", key="user_456", value={"item": "mouse", "price": 49})
# → hash("user_456") % 3 = partition 0
```

**Rule:** Same key → same partition → ordered. Different keys → possibly different partitions → no ordering guarantee between them.

**When to use a key:** When you need ordering per entity. E.g., all events for one user, one order, one account.

**When to skip the key (null):** When you don't care about ordering and just want maximum throughput via even distribution.

**⚠️ Common mistake: hot partitions.** If 80% of your traffic is for 10 popular users, and you key by `user_id`, those 10 users' partitions get hammered while others sit idle. This is the same **hotspot problem** from sharding (D21). If one key dominates, consider a compound key or a different partitioning strategy.

### Producer Batching [IMPORTANT]

The producer doesn't send every message individually. It **batches** messages destined for the same partition:

| Config | What it controls | Default |
|---|---|---|
| `batch.size` | Max bytes per batch | 16 KB |
| `linger.ms` | Max time to wait for more messages before sending | 0 ms (send immediately) |

With `linger.ms=0`, every message is sent as its own batch (low latency, low throughput).
With `linger.ms=5`, the producer waits up to 5ms to accumulate more messages into one batch (slightly higher latency, much higher throughput).

```
linger.ms=0:   [msg1] → send   [msg2] → send   [msg3] → send    (3 network calls)
linger.ms=5:   [msg1, msg2, msg3] → send                          (1 network call)
```

**Trade-off:** latency vs throughput. For most production systems, `linger.ms=5` and `batch.size=64KB` is a good starting point.

### Producer `acks` — The Durability Dial [CRITICAL]

This is the most important producer configuration. It controls how many replicas must confirm the write before the producer considers it successful.

| `acks` | What it means | Durability | Latency | When to use |
|---|---|---|---|---|
| `acks=0` | Producer doesn't wait for any confirmation | Messages can be lost (broker might crash before writing) | Fastest | Metrics, logs — loss is acceptable |
| `acks=1` | Leader writes to its log, confirms | Lost if leader crashes before followers replicate | Medium | Most use cases — good default |
| `acks=all` | Leader waits for ALL ISR replicas to write, then confirms | No data loss (unless entire ISR crashes) | Slowest | Payments, financial data, anything you can't afford to lose |

> **Real case: Data loss with acks=1**
> What happened: A data pipeline used `acks=1`. The leader broker accepted a batch of events, confirmed to the producer, then crashed before the follower replicated the data. The follower was promoted to leader — without those events. Data permanently lost.
> Root cause: `acks=1` means the leader confirms before replication completes. Leader crash in that window = data loss.
> Fix: Set `acks=all` + `min.insync.replicas=2`. Now the leader only confirms after at least 2 replicas (including itself) have the data. If only 1 replica is alive, writes are **rejected** rather than silently losing data.

The combination of `acks=all` + `min.insync.replicas=2` (with replication factor 3) is the production gold standard for critical data:

```
Replication factor = 3 (leader + 2 followers)
min.insync.replicas = 2

Write arrives → Leader writes → at least 1 follower writes → ACK to producer
               If only leader is alive (followers both dead):
               → Write REJECTED (not enough ISR members) → producer gets error
               → Better to fail loudly than silently lose data
```

---

## Putting It All Together — Full Write Path

```
Producer App                          Kafka Cluster
     │                                     │
     │  1. send(topic="orders",           │
     │     key="user_123",                │
     │     value={...})                   │
     │                                     │
     │  2. Serialize key+value to bytes   │
     │  3. hash("user_123") % 3 = 1      │
     │  4. Buffer in batch for P1         │
     │  5. linger.ms expires or batch full│
     │                                     │
     │ ──── batch of messages ──────────► │ Broker 1 (leader of P1)
     │                                     │  6. Append to P1 log file (sequential write)
     │                                     │  7. OS page cache holds it in RAM
     │                                     │  8. Follower on Broker 2 pulls and writes
     │                                     │  9. (if acks=all) wait for ISR confirmation
     │ ◄─────────── ACK ──────────────── │ 10. Confirm to producer
     │                                     │

```

---

## Trade-offs — When NOT to Use Kafka

| Scenario | Why not Kafka | Use instead |
|---|---|---|
| Simple task queue (email, resize image) | Kafka is overkill — no need for retention/replay | SQS, RabbitMQ, Redis |
| Low message volume (< 100/sec) | Kafka's operational overhead isn't justified | SQS (managed, zero ops) |
| Request-reply pattern | Kafka is pull-based, not designed for synchronous responses | Direct HTTP/gRPC call |
| Complex routing rules (message goes to different consumers based on content) | Kafka's routing is partition-based, not content-based | RabbitMQ (exchanges, routing keys) |

**Most common mistakes engineers make:**

1. **Using Kafka when SQS would suffice.** Kafka requires Zookeeper/KRaft, broker management, partition tuning. SQS is a managed API call. If you don't need replay or multi-consumer, don't pay Kafka's operational cost.

2. **Too few partitions at creation.** You can increase partitions later, but **messages with the same key will shift to different partitions** (because `hash(key) % new_N ≠ hash(key) % old_N`). This breaks ordering for in-flight consumers. Choose partitions thoughtfully upfront.

---

## When TO Use Kafka [CRITICAL]

| Need | Why Kafka wins |
|---|---|
| **Multiple independent consumers reading same events** | Each consumer group has its own offset — producer writes once, N consumers read independently. Adding a new consumer = zero producer changes. |
| **Replay / reprocessing after bugs** | Messages retained for days/weeks. Reset consumer offset → reprocess everything. Data is still on disk. |
| **High throughput (100K+ msg/sec)** | Sequential I/O + page cache + zero-copy. Nothing else comes close. |
| **Event sourcing / audit log** | Append-only immutable log = natural fit. Reconstruct any past state by replaying from offset 0. |
| **Backpressure — fast producer, slow consumer** | Disk-backed partitions buffer terabytes. Consumer reads at its own pace. No RAM pressure like RabbitMQ. |

**One-line answer:** Use Kafka when multiple systems need to independently consume the same high-throughput stream of events, especially if you'll ever need to replay them. Use a simple queue for everything else.

---

## Follow-up: How Backpressure Actually Works in Kafka

Producer writes 10K msg/sec. Consumer processes 2K msg/sec. The partition on disk absorbs the gap:

```
Producer: 10K msg/sec                    Consumer: 2K msg/sec
         │                                        │
         ▼                                        ▼
   ┌─────────────────────────────────────────────────┐
   │              Kafka Partition (on disk)           │
   │                                                  │
   │  [0][1][2][3]...............[9999][10000][10001]  │
   │                                                  │
   │   writes happen here →→→→→→→→→→→→→             │
   │             ↑                                    │
   │      consumer is here                            │
   │      (offset 2000)                               │
   │                                                  │
   │   Gap = 8,000 messages buffered on disk          │
   └─────────────────────────────────────────────────┘
```

| Time | Producer writes | Consumer reads | Lag (buffered) |
|---|---|---|---|
| t=1 | 10,000 | 2,000 | 8,000 |
| t=2 | 10,000 | 2,000 | 16,000 |
| t=3 | 10,000 | 2,000 | 24,000 |
| ...flash sale ends... | | | |
| t=10 | 0 | 2,000 | shrinking |
| t=22 | 0 | 2,000 | 0 — caught up ✓ |

**Why Kafka handles this but RabbitMQ doesn't:** RabbitMQ buffers unprocessed messages in **RAM**. Millions of messages = RAM full → broker triggers `memory alarm` → **blocks producers** or crashes. Kafka buffers on **disk** (sequential writes). A partition can hold terabytes. Consumer lag costs disk space, not RAM.

**Pull model enables natural backpressure:** Consumer calls `poll()` and decides how many messages to fetch. If its DB is under load, it pulls less. If DB recovers, it pulls more. No coordination needed.

---

## Follow-up: Consumer Parallelism with Partitions (Preview of D37)

**One consumer → all partitions (slow):**

```
Topic "payments" (4 partitions)

   P0: [msg][msg][msg]──┐
   P1: [msg][msg]───────┤──→  Consumer Instance 1 (handles ALL 4)
   P2: [msg][msg][msg]──┤     bottleneck — one process doing all the work
   P3: [msg]────────────┘
```

**Multiple consumers in same consumer group → partitions split (fast):**

```
Topic "payments" (4 partitions)

   P0: [msg][msg][msg]──→  Consumer Instance 1  (only P0)
   P1: [msg][msg]───────→  Consumer Instance 2  (only P1)
   P2: [msg][msg][msg]──→  Consumer Instance 3  (only P2)
   P3: [msg]────────────→  Consumer Instance 4  (only P3)

   4x throughput — true parallelism
```

**Hard rule: max useful consumers = number of partitions.**

| Partitions | Consumer instances | Result |
|---|---|---|
| 4 | 1 | 1 handles all 4 — slow |
| 4 | 2 | Each handles 2 — 2x speed |
| 4 | 4 | Each handles 1 — max speed |
| 4 | **5** | 4 active, **5th sits idle** — wasted |

**For payments with `key=user_id`:** alice's events all land on the same partition → same consumer → in order. bob's events on a different partition → different consumer → processed in parallel. Per-user ordering + cross-user parallelism.

```
P0: [alice: create] → [alice: charge] → [alice: receipt]  → Consumer 1 (in order ✓)
P1: [bob: create] → [bob: charge]                         → Consumer 2 (in order ✓)
P2: [carol: create] → [carol: charge] → [carol: receipt]  → Consumer 3 (in order ✓)
```

Consumer groups, rebalancing, and what happens when a consumer dies → covered in D37.

---

## Quiz + Answers

**Q1.** You have a topic `payments` with 4 partitions. You send all payment events with `key = user_id`. User `alice` has 3 events: create_order, charge_card, send_receipt. Are these 3 events guaranteed to be in order? Why or why not?

**A1.** Yes, they are guaranteed to be in order. Same key (`alice`) → `hash("alice") % 4` always produces the same partition number → all 3 events land in the same partition → ordering within one partition is strict and guaranteed. If the key were different for each event (or null), they could land on different partitions and ordering would NOT be guaranteed.

---

**Q2.** You configure your producer with `acks=all`, `replication_factor=3`, `min.insync.replicas=2`. One broker dies, then a second broker dies. What happens when the producer tries to send a message?

**A2.** The write is **rejected**. With only 1 broker alive, only 1 replica (the leader) can acknowledge. But `min.insync.replicas=2` requires at least 2 replicas to confirm before the write is considered successful. Kafka refuses the write and the producer gets a `NotEnoughReplicasException`. This is by design — better to fail loudly and alert ops than to silently accept writes that can't be durably replicated.

---

**Q3.** Your Kafka cluster is writing 500K messages/sec to disk. A colleague says "that's impossible, disk is way too slow for 500K writes/sec." Explain why Kafka can actually do this.

**A3.** Three reasons: (1) **Sequential I/O** — Kafka only appends to the end of log files, never random writes. Sequential disk writes can hit 600+ MB/sec, faster than random SSD writes. (2) **OS page cache** — recent writes stay in RAM automatically. Consumers reading recent data never hit disk. (3) **Zero-copy** — Kafka uses `sendfile()` to transfer data from page cache directly to the network card, bypassing user-space entirely. Kafka's speed comes from doing less, not more — the OS handles the heavy lifting.
