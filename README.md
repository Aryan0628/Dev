# Production Dev Learning Journey

A 70-day, one-topic-a-day roadmap to learn how production backend systems actually work — from the bytes on the wire to scaling a service to millions of users.

Each session is taught conversationally with Claude, then distilled into a cheatsheet I can come back to later.

---

## How This Works

**One topic per day. One hour per session.** Each session follows the same structure:

1. **Why it matters** — the real-world problem
2. **Core concept** — explained with analogies and diagrams
3. **How it works internally** — what happens under the hood
4. **Code / config example** — minimal real snippet
5. **Trade-offs** — when NOT to use it
6. **Quiz** — to verify I actually understood

After each session, a cheatsheet is saved in [`sessions/`](sessions/) so I can grep my way back to any concept.

---

## Progress

**Current day:** D10
**Streak:** 9 days
**Last completed:** D09 — Auth Fundamentals (session vs JWT vs OAuth 2.0)

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

See [CLAUDE.md](CLAUDE.md) for the full 70-day roadmap across 7 phases.

---

## The 7 Phases

| Phase | Topic | Days |
|---|---|---|
| 1 | Foundations — how the web works, core backend concepts | D01–D13 |
| 2 | Databases — SQL, Postgres internals, replication, sharding | D14–D28 |
| 3 | Caching & Queues — CDNs, Redis, Kafka | D29–D35 |
| 4 | System Design Patterns — scaling, microservices, distributed | D36–D48 |
| 5 | Observability — logging, metrics, tracing, alerting | D49–D55 |
| 6 | Security — auth, OWASP, secrets, network security | D56–D61 |
| 7 | DevOps & Cloud — Docker, K8s, CI/CD, AWS | D62–D70 |

---

## Repo Layout

```
Dev/
├── CLAUDE.md       # full roadmap + teaching format + progress tracker
├── README.md       # this file
└── sessions/       # one cheatsheet per day
    ├── D01-how-the-internet-works.md
    ├── D02-client-server-architecture.md
    ├── D03-http-versions.md
    ├── D04-tls-https.md
    ├── D05-rest-api-design.md
    ├── D06-processes-threads-async.md
    ├── D07-web-server-nginx.md
    ├── D08-load-balancing.md
    └── D09-auth-fundamentals.md
```

---

## Why I'm Doing This

Building real systems requires knowing what's *underneath* the framework you reach for — why the page is slow, why the DB locks up, why the deploy broke at 3am. This repo is my path from "I can build a CRUD app" to "I understand why the CRUD app falls over at 10k users — and how to fix it."

Cheatsheets are written in my own words. If something here is wrong, it's because I haven't learned it yet — open an issue or yell at me.
