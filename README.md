# Production Dev Learning Journey

A 76-day, one-topic-a-day roadmap to learn how production backend systems actually work — from the bytes on the wire to scaling a service to millions of users.

Each session is taught conversationally with Claude, then distilled into a cheatsheet I can come back to later.

---

## How This Works

**One topic per day. ~1.5 hours per session.** Each session follows the same structure:

1. **Why it matters** — the real-world problem; anchored with a real incident or failure
2. **Core concept** — explained with analogies and diagrams
3. **How it works internally** — what happens under the hood, failure modes woven in
4. **Code / config example** — minimal real snippet
5. **Trade-offs & when NOT to use it** — common production mistakes
6. **Quiz** — 3 questions + answers

After each session, a cheatsheet is saved in [`sessions/`](sessions/) so I can grep my way back to any concept.

---

## Progress

**Current day:** D37
**Streak:** 20 days
**Last completed:** D36 — Kafka Internals Part 1 (2026-06-12)

### Completed

| Day | Topic | Cheatsheet |
|---|---|---|
| D01 | How the Internet Works (DNS, TCP/IP, TLS, HTTP) | [D01](sessions/D01-how-the-internet-works.md) |
| D02 | Client-Server Architecture + Request-Response | [D02](sessions/D02-client-server-architecture.md) |
| D03 | HTTP/1.1 vs HTTP/2 vs HTTP/3 — what changed and why | [D03](sessions/D03-http-versions.md) |
| D04 | TLS/HTTPS — handshake, certificates, why it matters | [D04](sessions/D04-tls-https.md) |
| D05 | REST API Design — methods, status codes, idempotency | [D05](sessions/D05-rest-api-design.md) |
| D06 | Processes vs Threads vs Async (event loop, blocking vs non-blocking) | [D06](sessions/D06-processes-threads-async.md) |
| D07 | How a Web Server Works (Nginx internals, reverse proxy) | [D07](sessions/D07-web-server-nginx.md) |
| D08 | Load Balancing — L4 vs L7, algorithms (round robin, least conn, sticky) | [D08](sessions/D08-load-balancing.md) |
| D09 | Auth Fundamentals — session vs JWT vs OAuth 2.0 | [D09](sessions/D09-auth-fundamentals.md) |
| D10 | API Gateway — rate limiting, auth at the edge | [D10](sessions/D10-api-gateway.md) |
| D11 | WebSockets vs Long Polling vs SSE | [D11](sessions/D11-websockets-sse-long-polling.md) |
| D12 | API Versioning — evolving APIs without breaking clients | [D12](sessions/D12-api-versioning.md) |
| D13 | Revision Day 1 — D06–D12 revisit | [D13](sessions/D13-revision-day-1.md) |
| D14 | Revision Day 2 — D01–D05 + full Phase 1 quiz sweep | [D14](sessions/D14-revision-day-2.md) |
| D15 | SQL Fundamentals — indexes, EXPLAIN, N+1 problem | [D15](sessions/D15-sql-fundamentals.md) |
| D26 | Redis Internals — data structures, persistence (RDB vs AOF) | [D26](sessions/D26-redis-internals.md) |
| D32 | Caching Strategies — cache-aside, write-through, write-behind | [D32](sessions/D32-caching-strategies.md) |
| D33 | Cache Invalidation — TTL, event-driven, cache stampede | [D33](sessions/D33-cache-invalidation.md) |
| D34 | CDN Deep Dive — edge caching, cache-control headers, purging | [D34](sessions/D34-cdn-deep-dive.md) |
| D35 | Message Queues — async, at-least-once vs exactly-once | [D35](sessions/D35-message-queues-overview.md) |
| D36 | Kafka Internals Part 1 — topics, partitions, offsets, producers | [D36](sessions/D36-kafka-internals-part1.md) |

See [CLAUDE.md](CLAUDE.md) for the full 76-day roadmap across 7 phases.

---

## The 7 Phases

| Phase | Topic | Days | Status |
|---|---|---|---|
| 1 | Foundations — how the web works, core backend concepts | D01–D14 | ✅ 14/14 |
| 2 | Databases — SQL, Postgres internals, replication, sharding | D15–D31 | 2/17 |
| 3 | Caching & Queues — CDNs, Redis, Kafka | D32–D39 | 5/8 |
| 4 | System Design Patterns — scaling, microservices, distributed | D40–D53 | 0/14 |
| 5 | Observability — logging, metrics, tracing, alerting | D54–D60 | 0/7 |
| 6 | Security — auth, OWASP, secrets, network security | D61–D66 | 0/6 |
| 7 | DevOps & Cloud — Docker, K8s, CI/CD, AWS | D67–D76 | 0/10 |
| | **Total** | **D01–D76** | **21/76** |

---

## Repo Layout

```
Dev/
├── CLAUDE.md            # full roadmap + teaching format + progress tracker
├── README.md            # this file
├── personalization.md   # learning pattern + response crafting rules
└── sessions/            # one cheatsheet per day
    ├── D01-how-the-internet-works.md
    ├── D02-client-server-architecture.md
    ├── D03-http-versions.md
    ├── D04-tls-https.md
    ├── D05-rest-api-design.md
    ├── D06-processes-threads-async.md
    ├── D07-web-server-nginx.md
    ├── D08-load-balancing.md
    ├── D09-auth-fundamentals.md
    ├── D10-api-gateway.md
    ├── D11-websockets-sse-long-polling.md
    ├── D12-api-versioning.md
    ├── D13-revision-day-1.md
    ├── D14-revision-day-2.md
    ├── D15-sql-fundamentals.md
    ├── D26-redis-internals.md
    ├── D32-caching-strategies.md
    ├── D33-cache-invalidation.md
    ├── D34-cdn-deep-dive.md
    ├── D35-message-queues-overview.md
    └── D36-kafka-internals-part1.md
```

---

## Why I'm Doing This

Building real systems requires knowing what's *underneath* the framework you reach for — why the page is slow, why the DB locks up, why the deploy broke at 3am. This repo is my path from "I can build a CRUD app" to "I understand why the CRUD app falls over at 10k users — and how to fix it."

Cheatsheets are written in my own words. If something here is wrong, it's because I haven't learned it yet — open an issue or yell at me.
