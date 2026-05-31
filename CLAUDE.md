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

## Phase 1 — Foundations (D01–D13)

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
- [ ] D13 — **Revision Day** — revisit D06–D12, redo any quiz questions you got wrong

---

## Phase 2 — Databases (D14–D28)

- [ ] D14 — SQL fundamentals for production (indexes, EXPLAIN, N+1 problem)
- [ ] D15 — PostgreSQL internals — MVCC, WAL, vacuum
- [ ] D16 — Indexing deep dive — B-Tree, composite, partial, covering indexes
- [ ] D17 — SQL vs NoSQL — when to pick which and why
- [ ] D18 — CAP theorem / PACELC — consistency vs availability trade-offs *(new — foundational for all distributed systems)*
- [ ] D19 — Database replication — primary-replica, replication lag, read scaling
- [ ] D20 — Sharding — horizontal partitioning, shard keys, hotspots
- [ ] D21 — Consistent hashing — how it solves sharding and caching problems *(new)*
- [ ] D22 — Transactions Part 1 — ACID properties, isolation levels
- [ ] D23 — Transactions Part 2 — deadlocks, practical patterns, when to avoid transactions *(split — too much for one session)*
- [ ] D24 — Connection pooling — why it matters, PgBouncer
- [ ] D25 — Redis internals — data structures, persistence (RDB vs AOF)
- [ ] D26 — Multi-region databases — read replicas, active-active
- [ ] D27 — Database migrations — zero-downtime migrations, expand-contract pattern *(new)*

### Phase 2 Wrap-up
- [ ] D28 — **Revision Day** — revisit D14–D27, focus on CAP theorem + isolation levels

---

## Phase 3 — Caching & Queues (D29–D35)

- [ ] D29 — Caching strategies — cache-aside, write-through, write-behind
- [ ] D30 — Cache invalidation — TTL, event-driven, cache stampede / thundering herd
- [ ] D31 — CDN deep dive — edge caching, cache-control headers, purging
- [ ] D32 — Message queues overview — why async matters, at-least-once vs exactly-once
- [ ] D33 — Kafka internals Part 1 — topics, partitions, offsets, producers
- [ ] D34 — Kafka internals Part 2 — consumer groups, retention, rebalancing *(split — consumer groups alone need a full session)*

### Phase 3 Wrap-up
- [ ] D35 — **Revision Day** — cache invalidation + Kafka consumer groups are the hardest parts here

---

## Phase 4 — System Design Patterns (D36–D48)

- [ ] D36 — Scaling fundamentals — vertical vs horizontal, stateless services
- [ ] D37 — Zero to million users — how a system evolves step by step
- [ ] D38 — Microservices vs Monolith — trade-offs, when to split
- [ ] D39 — gRPC vs REST — protobuf, streaming, when to prefer gRPC
- [ ] D40 — Event-driven architecture — pub/sub, event sourcing basics
- [ ] D41 — Idempotency — why it matters, how to implement it
- [ ] D42 — Distributed transactions Part 1 — 2PC (Two-Phase Commit), where it breaks
- [ ] D43 — Distributed transactions Part 2 — Saga pattern, choreography vs orchestration *(split — architecturally very different from 2PC)*
- [ ] D44 — Rate limiting — token bucket, sliding window, implementation
- [ ] D45 — Circuit breaker pattern — failure isolation, retry strategies
- [ ] D46 — Feature flags — rollout strategies, kill switches, A/B testing *(new)*
- [ ] D47 — Design a URL shortener (end-to-end system design session)

### Phase 4 Wrap-up
- [ ] D48 — **Revision Day** — Saga vs 2PC + circuit breaker are interview favourites, go deep

---

## Phase 5 — Observability (D49–D55)

- [ ] D49 — Logging for production — structured logs, log levels, what to log
- [ ] D50 — Metrics — RED method (Rate, Errors, Duration), Prometheus basics
- [ ] D51 — Distributed tracing — spans, trace IDs, OpenTelemetry
- [ ] D52 — Alerting — SLI/SLO/SLA, alert fatigue, runbooks
- [ ] D53 — Debugging in production — reading stack traces, profiling basics
- [ ] D54 — Incident response — on-call, postmortems, blameless culture *(new — what do you DO when alerts fire?)*

### Phase 5 Wrap-up
- [ ] D55 — **Revision Day** — SLI/SLO/SLA definitions and RED method

---

## Phase 6 — Security (D56–D61)

> Note: Auth fundamentals (JWT, OAuth) already covered in D09

- [ ] D56 — OWASP Part 1 — SQL injection, XSS, CSRF in depth *(split — 10 vectors in 1 hour = surface level only)*
- [ ] D57 — OWASP Part 2 — IDOR, broken auth, security misconfigurations, others
- [ ] D58 — Secrets management — env vars, Vault, never in code
- [ ] D59 — Network security — firewalls, VPC, private subnets, zero-trust
- [ ] D60 — Input validation & rate limiting for security

### Phase 6 Wrap-up
- [ ] D61 — **Revision Day** — OWASP top 10 from memory, then check

---

## Phase 7 — DevOps & Cloud (D62–D70)

- [ ] D62 — Docker Part 1 — images, containers, layers
- [ ] D63 — Docker Part 2 — multi-stage builds, networking, volumes, docker-compose *(split — too much for one session)*
- [ ] D64 — Kubernetes Part 1 — pods, deployments, why K8s exists
- [ ] D65 — Kubernetes Part 2 — services, ingress, configmaps, secrets *(split)*
- [ ] D66 — Kubernetes Part 3 — scaling, health checks, production patterns, when NOT to use K8s *(split)*
- [ ] D67 — CI/CD pipeline — build, test, deploy, rollback strategies
- [ ] D68 — Infrastructure as Code — Terraform basics
- [ ] D69 — Cloud fundamentals — AWS core services (EC2, S3, RDS, SQS, Lambda)

### Phase 7 Wrap-up
- [ ] D70 — **Revision Day** — K8s concepts + Terraform basics

---

## Progress Tracker

| Phase | Days | Status |
|---|---|---|
| Phase 1 — Foundations | D01–D13 | 13/13 done |
| Phase 2 — Databases | D14–D28 | 0/15 done |
| Phase 3 — Caching & Queues | D29–D35 | 0/7 done |
| Phase 4 — System Design | D36–D48 | 0/13 done |
| Phase 5 — Observability | D49–D55 | 0/7 done |
| Phase 6 — Security | D56–D61 | 0/6 done |
| Phase 7 — DevOps & Cloud | D62–D70 | 0/9 done |
| **Total** | **D01–D70** | **12/70 done** |

---

## Progress
- Current day: D13 — Revision Day (Phase 1 wrap-up)
- Last completed: D12 — API Versioning (2026-05-31)
- Streak: 13 days
