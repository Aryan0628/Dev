# D42 — Microservices vs Monolith: Trade-offs, When to Split

---

## Why It Matters — The Most Expensive Architecture Decision

This decision shapes deployment, team structure, failure propagation, and shipping speed. Getting it wrong in either direction costs years.

> **Real case: Twitter — 2009–2013, the "Fail Whale" era**
> What happened: Ruby on Rails monolith couldn't scale. A slow DB query in timeline brought down DMs, search, notifications — all in one process.
> Root cause: Everything coupled. Can't scale timeline independently. Can't deploy a fix to DMs without redeploying everything.
> Cost / impact: "Fail Whale" error page during every major world event.
> Fix: 3-year migration to microservices. Separate services for timeline, user, tweet, DM, notifications. Migration itself introduced new distributed debugging problems.

> **Real case: Etsy — chose to STAY monolith**
> What happened: While everyone migrated to microservices (2014–2016), Etsy kept their PHP monolith. Invested in tooling — 50+ deploys/day, feature flags, excellent monitoring.
> Result: Shipped faster than many microservices companies. Served billions in transactions. Proved that a well-managed monolith can outperform poorly-managed microservices.

---

## What Is a Monolith? [CRITICAL]

Single deployable unit — one codebase, one process, one deploy artifact.

```
┌──────────────────────────────────────────┐
│              Monolith App                 │
│                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │  Users   │ │  Orders  │ │ Payments │ │
│  │  Module  │ │  Module  │ │  Module  │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ │
│       └─────────────┴─────────────┘       │
│              Shared Database              │
│  One process. One deploy. One DB.        │
└──────────────────────────────────────────┘
```

Modules call each other via **function calls** — same process, nanoseconds, never fails.

## What Are Microservices? [CRITICAL]

Separate, independently deployable services, each owning one business capability and its own data.

```
┌───────────┐     ┌───────────┐     ┌───────────┐
│   Users   │     │  Orders   │     │ Payments  │
│  Service  │     │  Service  │     │  Service  │
│  ┌─────┐  │     │  ┌─────┐  │     │  ┌─────┐  │
│  │ DB  │  │     │  │ DB  │  │     │  │ DB  │  │
│  └─────┘  │     │  └─────┘  │     │  └─────┘  │
└─────┬─────┘     └─────┬─────┘     └─────┬─────┘
      └────── network calls (HTTP/gRPC/Kafka) ──────┘
```

Services call each other via **network** — milliseconds, **can fail**.

**Fundamental shift:** function call (nanoseconds, never fails) → network call (milliseconds, can fail).

---

## Monolith Advantages

| Advantage | Why |
|---|---|
| **Simple to develop** | One codebase, one IDE, one debug session |
| **Simple to deploy** | Build one artifact, deploy once |
| **Simple to test** | One test suite, no network mocks |
| **No network overhead** | Function calls, not HTTP calls |
| **No distributed failures** | If it's running, everything works |
| **Easy data joins** | One DB → SQL JOIN across any tables |
| **Easy transactions** | One DB → ACID transactions across everything |

## Microservices Advantages

| Advantage | Why |
|---|---|
| **Independent deployment** | Deploy Orders without touching Users |
| **Independent scaling** | Scale only what's hot |
| **Team independence** | Teams own services end-to-end, ship independently |
| **Tech flexibility** | Each service picks its own stack |
| **Fault isolation** | One service dies, others keep running |
| **Smaller codebases** | New engineer ramps up in days, not months |

## What Microservices ADD (Operational Tax) [CRITICAL]

| What you gain | What you pay |
|---|---|
| Independent deployment | CI/CD per service (D72), containers (D69–71) |
| Independent scaling | Service discovery, LB per service (D08) |
| Fault isolation | Circuit breakers (D49), retries, timeouts |
| Team independence | API contracts, versioning (D12), backward compatibility |
| Tech flexibility | Polyglot monitoring, shared logging (D54) |
| Smaller codebases | Distributed tracing (D56) for cross-service debugging |
| — | API Gateway (D10) as front door |
| — | Message queues (D35–37) for async communication |
| — | **No SQL JOINs across services** |
| — | **No ACID transactions across services** — need Saga (D47) |

**One-line answer:** Microservices trade code complexity for operational complexity. Simpler code, 10x the infrastructure.

---

## Communication Between Services [CRITICAL]

### Synchronous — Request/Response (HTTP or gRPC)

```
Orders Service
    ├── GET /users/123 ────→ Users Service (wait for response)
    ├── POST /payments ────→ Payments Service (wait for response)
    └── Return to user
```

Problem: If Payments is slow/down, Orders is stuck/fails. Synchronous coupling (D35).

### Asynchronous — Event-Driven (Kafka)

```
Orders Service
    ├── Save order (status: "pending")
    ├── Publish to Kafka: "order_created"
    └── Return to user immediately

Kafka topic: "order-events"
    ├──→ Payments Service (consumer group) — charges card
    ├──→ Inventory Service (consumer group) — reserves stock
    └──→ Notification Service (consumer group) — sends email
```

Advantage: No waiting, no cascade failures, backpressure handled by Kafka (D36).
Disadvantage: Eventual consistency. User gets "Order placed!" before payment is charged.

**Most real systems use BOTH.** Sync for reads, async for writes/events.

---

## The Distributed Monolith — Worst of Both Worlds [CRITICAL]

```
❌ Distributed Monolith

┌───────────┐  ┌───────────┐  ┌───────────┐
│   Users   │  │  Orders   │  │ Payments  │
│  Service  │  │  Service  │  │  Service  │
└─────┬─────┘  └─────┬─────┘  └─────┬─────┘
      └──────────────┬┴──────────────┘
              ┌──────▼──────┐
              │  SHARED DB  │  ← all services query all tables
              └─────────────┘     schema change breaks everyone
```

**Signs you have one:**
1. Services share a database
2. Synchronous chain: A → B → C → D (one failure cascades)
3. Can't deploy one service without deploying others
4. Teams coordinate releases

**Fix:** Each service owns its data. Need user data from Orders? Call Users API, don't query Users table.

---

## When to Use Which [CRITICAL]

### Start Monolith When:

| Signal | Why |
|---|---|
| Small team (< 10 engineers) | Can't afford microservices operational overhead |
| New product, unclear boundaries | Splitting wrong is worse than not splitting |
| Speed of iteration matters | Monolith ships faster early |
| Low scale (< 10K req/sec) | One process handles this |

### Split to Microservices When:

| Signal | Why |
|---|---|
| Teams stepping on each other | Merge conflicts, deploy coordination |
| One component needs different scaling | Search needs 50 instances; profiles need 3 |
| Deploy frequency bottlenecked | "Can't deploy my fix, another team's code broken" |
| Different reliability requirements | Payment 99.99%; recommendation 99.9% |
| Codebase too large to reason about | New engineers take months to understand |

### The Golden Rule

> **"Don't start with microservices."** — Martin Fowler
>
> Build monolith → understand domain → find natural boundaries → extract services **one at a time** when specific pain justifies it.

```
Year 1:   Monolith (5 engineers, ship fast)
Year 2:   Monolith (15 engineers, starting to feel pain)
Year 2.5: Extract Payment service (compliance/scaling needs)
Year 3:   Extract Search (needs Elasticsearch)
Year 4:   3 services + monolith for the rest
Year 5+:  Gradually extract more as team grows to 100+
```

**One-line answer:** Start monolith, split when you feel specific pain, never split just because "everyone does microservices."

---

## How to Find Service Boundaries [IMPORTANT]

**Good boundary = one team, one service, one DB, one business capability.**

```
✅ Good: Orders Service owns all order logic. Users Service owns all user logic.
         Orders needs user name? → calls Users API.
         Changing order schema doesn't affect Users.

❌ Bad:  "Order-Creation Service" + "Order-Update Service"
         Both need same tables, same data, same transactions.
         This is a monolith with extra network hops.
```

**Quick test:** Can you deploy Service A without telling Service B's team? Yes → good boundary. No → too coupled.

---

## Common Mistakes

1. **"Netflix does it, so should we."** Netflix has 2000+ engineers and a platform team. You have 8. Monolith ships 10x faster at your scale.

2. **Splitting too early.** Create "Notifications Service" on day 1. Six months later, it's deeply coupled to orders, users, payments. Distributed monolith harder to fix than if you'd kept the monolith.

3. **Sharing a database across services.** #1 cause of distributed monoliths. Two services querying the same table = one service pretending to be two.

4. **Synchronous chains without fallbacks.** A → B → C → D, all sync HTTP. D slow → cascade timeout. Need circuit breakers (D49) and async (D35–37).

---

## Quiz + Answers

**Q1.** 6 engineers, e-commerce site, 5K req/sec. Senior engineer proposes 8 microservices from day one "to be scalable." Your response?

**A1.** Push back. At 6 engineers and 5K req/sec, monolith is right. (1) 6 engineers can't manage 8 services — need CI/CD, distributed tracing, service discovery, containers per service. More infra work than product work. (2) 5K req/sec handled easily by one monolith. (3) Unclear domain = wrong boundaries = distributed monolith. Start monolith, extract when specific pain justifies it.

---

**Q2.** 50-engineer monolith. Payments bug brought down entire app including product catalog. Orders team blocked by Payments team's failing tests. What architectural change?

**A2.** Extract Payments into its own microservice. (1) Fault isolation — Payments crash only affects Payments, not catalog. (2) Deploy independence — separate CI/CD, separate deploys. (3) Own database — payment data has different compliance needs. Communication: Orders publishes `order_created` to Kafka, Payments consumes asynchronously. If Payments down, events buffer in Kafka.

---

**Q3.** Split monolith into Users, Orders, Payments services. All three still read/write the same PostgreSQL database. What's wrong?

**A3.** Distributed monolith. Coupled through shared DB — schema change forces coordinated deploys across all services. Added network latency and distributed failures with zero independence gained. Fix: each service gets its own database. Orders needs user data → calls Users API, not Users table directly. Core rule: **each service owns its data**.
