# Production Dev Learning Roadmap

## Teaching Format
- Each session = ~1.5 hours, one topic at a time (going a bit over is fine)
- Structure per session — no fixed time per section, flow naturally:
  1. **Why it matters** — real-world problem it solves; anchor with a real incident or failure that happened because someone got this wrong
  2. **Core concept** — explained with diagrams in text / analogies
  3. **How it works internally** — what happens under the hood; weave in failure modes and what breaks when the internals are misunderstood
  4. **Code / config example** — minimal real snippet
  5. **Trade-offs & when NOT to use it** — include the most common mistakes engineers make in production with this concept
  6. **3 quiz questions + answers** — you answer, I correct; answers always included in cheatsheet
- Real incidents and failure modes are not separate sections — they are woven into the relevant part of the session where the concept is taught
- Archon is a supplementary lookup only; own knowledge + web are primary
- Track progress: mark [x] when done

## Teaching Style Notes
- Explain ALL new terms inline — assume nothing is obvious
- Go deep on concepts when asked, never rush
- Use analogies before technical definitions
- Every session saves a cheatsheet in sessions/Dxx-topic.md
- **Read `personalization.md` before every session** — it tracks Aryan's learning pattern and how to craft better responses

---

## Phase 1 — Foundations (D01–D14)

### How the Web Works
- [x] D01 — How the Internet Works (DNS, TCP/IP, HTTP lifecycle)
- [x] D02 — Client-Server Architecture + Request-Response model
- [x] D03 — HTTP/1.1 vs HTTP/2 vs HTTP/3 — what changed and why
- [x] D04 — TLS/HTTPS — handshake, certificates, why it matters
- [x] D05 — REST API design — methods, status codes, idempotency

### Core Backend Concepts
- [x] D06 — Processes vs Threads vs Async (event loop, blocking vs non-blocking)
- [x] D07 — How a web server works (Nginx internals, reverse proxy)
- [x] D08 — Load Balancing — L4 vs L7, algorithms (round robin, least conn, sticky)
- [x] D09 — Auth fundamentals — session vs JWT vs OAuth 2.0 *(moved up from Security — needed before API Gateway makes full sense)*
- [x] D10 — API Gateway — what it is, rate limiting, auth at the edge
- [x] D11 — WebSockets vs Long Polling vs SSE — when to use which
- [x] D12 — API Versioning — how to evolve APIs without breaking clients *(new)*

### Phase 1 Wrap-up
- [ ] D13 — **Revision Day 1** — revisit D06–D12, redo any quiz questions you got wrong
- [ ] D14 — **Revision Day 2** — revisit D01–D05, full Phase 1 quiz sweep

---

## Phase 2 — Databases (D15–D31)

- [ ] D15 — SQL fundamentals for production (indexes, EXPLAIN, N+1 problem)
- [ ] D16 — PostgreSQL internals — MVCC, WAL, vacuum
- [ ] D17 — Indexing deep dive — B-Tree, composite, partial, covering indexes
- [ ] D18 — SQL vs NoSQL — when to pick which and why
- [ ] D19 — CAP theorem / PACELC — consistency vs availability trade-offs *(new — foundational for all distributed systems)*
- [ ] D20 — Database replication — primary-replica, replication lag, read scaling
- [ ] D21 — Sharding — horizontal partitioning, shard keys, hotspots
- [ ] D22 — Consistent hashing — how it solves sharding and caching problems *(new)*
- [ ] D23 — Transactions Part 1 — ACID properties, isolation levels
- [ ] D24 — Transactions Part 2 — deadlocks, practical patterns, when to avoid transactions *(split — too much for one session)*
- [ ] D25 — Connection pooling — why it matters, PgBouncer
- [ ] D26 — Redis internals — data structures, persistence (RDB vs AOF)
- [ ] D27 — Multi-region databases — read replicas, active-active
- [ ] D28 — Database migrations — zero-downtime migrations, expand-contract pattern *(new)*

### Phase 2 Wrap-up
- [ ] D29 — **Revision Day 1** — CAP theorem + isolation levels (most interview-critical)
- [ ] D30 — **Revision Day 2** — sharding, consistent hashing, replication lag
- [ ] D31 — **Revision Day 3** — MVCC, ACID, connection pooling + full Phase 2 quiz sweep

---

## Phase 3 — Caching & Queues (D32–D39)

- [ ] D32 — Caching strategies — cache-aside, write-through, write-behind
- [ ] D33 — Cache invalidation — TTL, event-driven, cache stampede / thundering herd
- [ ] D34 — CDN deep dive — edge caching, cache-control headers, purging
- [ ] D35 — Message queues overview — why async matters, at-least-once vs exactly-once
- [ ] D36 — Kafka internals Part 1 — topics, partitions, offsets, producers
- [ ] D37 — Kafka internals Part 2 — consumer groups, retention, rebalancing *(split — consumer groups alone need a full session)*

### Phase 3 Wrap-up
- [ ] D38 — **Revision Day 1** — cache invalidation + Kafka consumer groups (hardest parts)
- [ ] D39 — **Revision Day 2** — full Phase 3 quiz sweep

---

## Phase 4 — System Design Patterns (D40–D53)

- [ ] D40 — Scaling fundamentals — vertical vs horizontal, stateless services
- [ ] D41 — Zero to million users — how a system evolves step by step
- [ ] D42 — Microservices vs Monolith — trade-offs, when to split
- [ ] D43 — gRPC vs REST — protobuf, streaming, when to prefer gRPC
- [ ] D44 — Event-driven architecture — pub/sub, event sourcing basics
- [ ] D45 — Idempotency — why it matters, how to implement it
- [ ] D46 — Distributed transactions Part 1 — 2PC (Two-Phase Commit), where it breaks
- [ ] D47 — Distributed transactions Part 2 — Saga pattern, choreography vs orchestration *(split — architecturally very different from 2PC)*
- [ ] D48 — Rate limiting — token bucket, sliding window, implementation
- [ ] D49 — Circuit breaker pattern — failure isolation, retry strategies
- [ ] D50 — Feature flags — rollout strategies, kill switches, A/B testing *(new)*
- [ ] D51 — Design a URL shortener (end-to-end system design session)

### Phase 4 Wrap-up
- [ ] D52 — **Revision Day 1** — Saga vs 2PC + circuit breaker (most interview-critical)
- [ ] D53 — **Revision Day 2** — full Phase 4 quiz sweep

---

## Phase 5 — Observability (D54–D60)

- [ ] D54 — Logging for production — structured logs, log levels, what to log
- [ ] D55 — Metrics — RED method (Rate, Errors, Duration), Prometheus basics
- [ ] D56 — Distributed tracing — spans, trace IDs, OpenTelemetry
- [ ] D57 — Alerting — SLI/SLO/SLA, alert fatigue, runbooks
- [ ] D58 — Debugging in production — reading stack traces, profiling basics
- [ ] D59 — Incident response — on-call, postmortems, blameless culture *(new — what do you DO when alerts fire?)*

### Phase 5 Wrap-up
- [ ] D60 — **Revision Day** — SLI/SLO/SLA definitions and RED method

---

## Phase 6 — Security (D61–D66)

> Note: Auth fundamentals (JWT, OAuth) already covered in D09

- [ ] D61 — OWASP Part 1 — SQL injection, XSS, CSRF in depth *(split — 10 vectors in 1 hour = surface level only)*
- [ ] D62 — OWASP Part 2 — IDOR, broken auth, security misconfigurations, others
- [ ] D63 — Secrets management — env vars, Vault, never in code
- [ ] D64 — Network security — firewalls, VPC, private subnets, zero-trust
- [ ] D65 — Input validation & rate limiting for security

### Phase 6 Wrap-up
- [ ] D66 — **Revision Day** — OWASP top 10 from memory, then check

---

## Phase 7 — DevOps & Cloud (D67–D76)

- [ ] D67 — Docker Part 1 — images, containers, layers
- [ ] D68 — Docker Part 2 — multi-stage builds, networking, volumes, docker-compose *(split — too much for one session)*
- [ ] D69 — Kubernetes Part 1 — pods, deployments, why K8s exists
- [ ] D70 — Kubernetes Part 2 — services, ingress, configmaps, secrets *(split)*
- [ ] D71 — Kubernetes Part 3 — scaling, health checks, production patterns, when NOT to use K8s *(split)*
- [ ] D72 — CI/CD pipeline — build, test, deploy, rollback strategies
- [ ] D73 — Infrastructure as Code — Terraform basics
- [ ] D74 — Cloud fundamentals — AWS core services (EC2, S3, RDS, SQS, Lambda)

### Phase 7 Wrap-up
- [ ] D75 — **Revision Day 1** — K8s concepts (3 sessions worth, needs a full pass)
- [ ] D76 — **Revision Day 2** — Terraform + CI/CD + full Phase 7 quiz sweep

---

## Progress Tracker

| Phase | Days | Status |
|---|---|---|
| Phase 1 — Foundations | D01–D14 | 12/14 done |
| Phase 2 — Databases | D15–D31 | 0/17 done |
| Phase 3 — Caching & Queues | D32–D39 | 0/8 done |
| Phase 4 — System Design | D40–D53 | 0/14 done |
| Phase 5 — Observability | D54–D60 | 0/7 done |
| Phase 6 — Security | D61–D66 | 0/6 done |
| Phase 7 — DevOps & Cloud | D67–D76 | 0/10 done |
| **Total** | **D01–D76** | **12/76 done** |

---

## Progress
- Current day: D13 — Revision Day 1 (Phase 1 wrap-up)
- Last completed: D12 — API Versioning (2026-05-31)
- Streak: 13 days
