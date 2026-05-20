# Production Dev Learning Journey

A 50-day, one-topic-a-day roadmap to learn how production backend systems actually work — from the bytes on the wire to scaling a service to millions of users.

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

**Current day:** D03
**Streak:** 2 days
**Last completed:** D02 — Client-Server Architecture + Request-Response

### Completed

| Day | Topic | Cheatsheet |
|---|---|---|
| D01 | How the Internet Works (DNS, TCP/IP, TLS, HTTP) | [D01](sessions/D01-how-the-internet-works.md) |
| D02 | Client-Server Architecture + Request-Response | [D02](sessions/D02-client-server-architecture.md) |

See [CLAUDE.md](CLAUDE.md) for the full 50-day roadmap across 7 phases.

---

## The 7 Phases

| Phase | Topic | Days |
|---|---|---|
| 1 | Foundations — how the web works, core backend concepts | D01–D10 |
| 2 | Databases — SQL, Postgres internals, replication, sharding | D11–D20 |
| 3 | Caching & Queues — CDNs, Redis, Kafka | D21–D25 |
| 4 | System Design Patterns — scaling, microservices, distributed | D26–D35 |
| 5 | Observability — logging, metrics, tracing, alerting | D36–D40 |
| 6 | Security — auth, OWASP, secrets, network security | D41–D45 |
| 7 | DevOps & Cloud — Docker, K8s, CI/CD, AWS | D46–D50 |

---

## Repo Layout

```
Dev/
├── CLAUDE.md       # full roadmap + teaching format + progress tracker
├── README.md       # this file
└── sessions/       # one cheatsheet per day
    ├── D01-how-the-internet-works.md
    └── D02-client-server-architecture.md
```

---

## Why I'm Doing This

Building real systems requires knowing what's *underneath* the framework you reach for — why the page is slow, why the DB locks up, why the deploy broke at 3am. This repo is my path from "I can build a CRUD app" to "I understand why the CRUD app falls over at 10k users — and how to fix it."

Cheatsheets are written in my own words. If something here is wrong, it's because I haven't learned it yet — open an issue or yell at me.
