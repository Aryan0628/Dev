# Production Dev Learning Roadmap

## Teaching Format
- Each session = 1 hour, one topic at a time
- Structure per session:
  1. **Why it matters** (5 min) — real-world problem it solves
  2. **Core concept** (20 min) — explained with diagrams in text / analogies
  3. **How it works internally** (15 min) — what happens under the hood
  4. **Code / config example** (10 min) — minimal real snippet
  5. **Trade-offs & when NOT to use it** (5 min)
  6. **3 quiz questions + answers** (5 min) — you answer, I correct; answers always included in cheatsheet
- Archon is used as reference source, not copy-paste
- Track progress: mark [x] when done

## Teaching Style Notes
- Explain ALL new terms inline — assume nothing is obvious
- Go deep on concepts when asked, never rush
- Use analogies before technical definitions
- Every session saves a cheatsheet in sessions/Dxx-topic.md
- **Read `personalization.md` before every session** — it tracks Aryan's learning pattern and how to craft better responses

---

## Phase 1 — Foundations (Weeks 1–2)

### How the Web Works
- [x] D01 — How the Internet Works (DNS, TCP/IP, HTTP lifecycle)
- [x] D02 — Client-Server Architecture + Request-Response model
- [x] D03 — HTTP/1.1 vs HTTP/2 vs HTTP/3 — what changed and why
- [x] D04 — TLS/HTTPS — handshake, certificates, why it matters
- [x] D05 — REST API design — methods, status codes, idempotency

### Core Backend Concepts
- [ ] D06 — Processes vs Threads vs Async (event loop, blocking vs non-blocking)
- [ ] D07 — How a web server works (Nginx internals, reverse proxy)
- [ ] D08 — Load Balancing — L4 vs L7, algorithms (round robin, least conn, sticky)
- [ ] D09 — API Gateway — what it is, rate limiting, auth at the edge
- [ ] D10 — WebSockets vs Long Polling vs SSE — when to use which

---

## Phase 2 — Databases (Weeks 3–4)

- [ ] D11 — SQL fundamentals for production (indexes, EXPLAIN, N+1 problem)
- [ ] D12 — PostgreSQL internals — MVCC, WAL, vacuum
- [ ] D13 — Indexing deep dive — B-Tree, composite, partial, covering indexes
- [ ] D14 — SQL vs NoSQL — when to pick which and why
- [ ] D15 — Database replication — primary-replica, replication lag, read scaling
- [ ] D16 — Sharding — horizontal partitioning, shard keys, hotspots
- [ ] D17 — Transactions — ACID, isolation levels, deadlocks
- [ ] D18 — Connection pooling — why it matters, PgBouncer
- [ ] D19 — Redis internals — data structures, persistence (RDB vs AOF)
- [ ] D20 — Multi-region databases — read replicas, active-active

---

## Phase 3 — Caching & Queues (Week 5)

- [ ] D21 — Caching strategies — cache-aside, write-through, write-behind
- [ ] D22 — Cache invalidation — TTL, event-driven, cache stampede / thundering herd
- [ ] D23 — CDN deep dive — edge caching, cache-control headers, purging
- [ ] D24 — Message queues overview — why async matters, at-least-once vs exactly-once
- [ ] D25 — Kafka internals — partitions, offsets, consumer groups, retention

---

## Phase 4 — System Design Patterns (Weeks 6–7)

- [ ] D26 — Scaling fundamentals — vertical vs horizontal, stateless services
- [ ] D27 — Zero to million users — how a system evolves step by step
- [ ] D28 — Microservices vs Monolith — trade-offs, when to split
- [ ] D29 — gRPC vs REST — protobuf, streaming, when to prefer gRPC
- [ ] D30 — Event-driven architecture — pub/sub, event sourcing basics
- [ ] D31 — Idempotency — why it matters, how to implement it
- [ ] D32 — Distributed transactions — 2PC, saga pattern
- [ ] D33 — Rate limiting — token bucket, sliding window, implementation
- [ ] D34 — Circuit breaker pattern — failure isolation, retry strategies
- [ ] D35 — Design a URL shortener (end-to-end system design session)

---

## Phase 5 — Observability (Week 8)

- [ ] D36 — Logging for production — structured logs, log levels, what to log
- [ ] D37 — Metrics — RED method (Rate, Errors, Duration), Prometheus basics
- [ ] D38 — Distributed tracing — spans, trace IDs, OpenTelemetry
- [ ] D39 — Alerting — SLI/SLO/SLA, alert fatigue, runbooks
- [ ] D40 — Debugging in production — reading stack traces, profiling basics

---

## Phase 6 — Security (Week 9)

- [ ] D41 — Auth fundamentals — session vs JWT vs OAuth 2.0
- [ ] D42 — OWASP Top 10 — SQL injection, XSS, CSRF, IDOR
- [ ] D43 — Secrets management — env vars, Vault, never in code
- [ ] D44 — Network security — firewalls, VPC, private subnets, zero-trust
- [ ] D45 — Input validation & rate limiting for security

---

## Phase 7 — DevOps & Cloud (Week 10)

- [ ] D46 — Docker — images, containers, layers, multi-stage builds
- [ ] D47 — Kubernetes basics — pods, deployments, services, ingress
- [ ] D48 — CI/CD pipeline — build, test, deploy, rollback strategies
- [ ] D49 — Infrastructure as Code — Terraform basics
- [ ] D50 — Cloud fundamentals — AWS core services (EC2, S3, RDS, SQS, Lambda)

---

## Progress
- Current day: D06
- Last completed: D05 — REST API Design (2026-05-23)
- Streak: 5 days
