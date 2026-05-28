# D10 — API Gateway: What It Is, Rate Limiting, Auth at the Edge

**Date:** 2026-05-28
**Phase:** 1 — Foundations

---

## Why It Matters

Without a gateway, every client must know the address of every backend service, handle TLS for each, and re-implement auth and rate limiting per service. Every service re-implements the same security logic — dozens of places to get it wrong.

An API Gateway = one front door. Handles cross-cutting concerns once so backend services just do their actual job.

---

## Core Concept

**Analogy:** Hotel front desk. Guests don't wander looking for housekeeping and room service. They talk to the front desk — it routes to the right department.

```
              ┌─────────────────────────────────────────┐
              │             API GATEWAY                  │
Client ──────>│  1. TLS termination                      │──> user-service
              │  2. Auth check (JWT/API key)             │──> order-service
              │  3. Rate limit check                     │──> payment-service
              │  4. Route to correct upstream service    │──> etc.
              │  5. Log + trace the request              │
              └─────────────────────────────────────────┘
```

**Vocabulary:**
- **Upstream** = backend service behind the gateway (e.g. user-service)
- **Downstream** = client calling the gateway (e.g. mobile app)
- **Route** = rule: "requests matching `/users/*` go to user-service"
- **Plugin / Middleware** = function the gateway runs on every request (auth plugin, rate-limit plugin, etc.)
- **TLS termination** = TLS = Transport Layer Security (D04). Gateway handles the HTTPS handshake with the client. Internal traffic travels as plain HTTP — services don't need their own TLS certs.

---

## How It Works Internally

### The Request Pipeline

```
Client
  │
  ▼
[1] TLS termination         ← gateway decrypts HTTPS, handles cert
  │
  ▼
[2] Auth check              ← validate JWT or API key
  │   pass → continue
  │   fail → 401 Unauthorized, stop here
  ▼
[3] Rate limit check        ← is this client sending too many requests?
  │   under limit → continue
  │   over limit  → 429 Too Many Requests, stop here
  ▼
[4] Route matching          ← which upstream does this path belong to?
  │   /users/* → user-service:3001
  │   /orders/* → order-service:3002
  ▼
[5] Forward request upstream
  │   gateway rewrites headers, forwards HTTP (no TLS needed internally)
  ▼
[6] Get response from upstream
  ▼
[7] Transform response if needed (strip internal fields, etc.)
  ▼
Client gets response
```

If step 2 or 3 fails, the request **never reaches your backend**. Bad traffic rejected at the edge before wasting service resources.

---

### Auth at the Edge

```
Client sends:
  GET /orders/123
  Authorization: Bearer eyJhbGci...

Gateway does:
  1. Split token → header, payload, signature
  2. Recompute: HMAC_SHA256(header + "." + payload, GATEWAY_SECRET)
  3. Compare to received signature → match = valid
  4. Check exp (expiry timestamp) in payload → not expired?
  5. Decode payload → { user_id: 42 }

Gateway forwards to order-service:
  GET /orders/123
  X-User-Id: 42          ← gateway injects this
  X-User-Email: a@b.com  ← optional, from JWT payload
  (Authorization header stripped — not forwarded)

order-service receives request:
  reads X-User-Id: 42
  done. no JWT logic needed.
```

This is called **"auth offloading."** JWT validation lives in exactly one place. Backend services trust the gateway and just read injected headers.

**One-line summary:** Gateway validates the JWT, strips it, injects user identity as plain headers — backends trust those headers without knowing JWT exists.

---

### Rate Limiting at the Edge

**Token Bucket Algorithm:**

Analogy: bucket holds max 100 tokens, refills at 10/sec. Each request costs 1 token.

```
Client A's bucket: [████████████] 100 tokens
  → request comes in → consume 1 → 99 tokens
  → 100 requests/sec burst → bucket empties → 429 Too Many Requests

Meanwhile, tokens refill at 10/sec
  → 1 second passes → bucket goes from 0 to 10
  → client can make 10 more requests
```

Allows **controlled bursts** — better UX than a hard per-second cap.

---

### The Distributed Rate Limiting Problem

4 gateway instances behind a load balancer (D08). Each runs in its own process with its own memory.

```
┌──────────────────────────────────────────────┐
│              Load Balancer                   │
└──────┬──────────┬──────────┬──────────┬──────┘
       ▼          ▼          ▼          ▼
   Gateway1   Gateway2   Gateway3   Gateway4
   bucket=25  bucket=25  bucket=25  bucket=25
```

User sends 100 requests. Load balancer spreads 25 to each. Every gateway sees 25 — under the 100 limit. User gets 400 req/min through the system. **In-memory counters are wrong.**

**Fix: shared Redis counter**

```
Gateway1 ──── INCR user:42:count ───┐
Gateway2 ──── INCR user:42:count ───┤──> Redis
Gateway3 ──── INCR user:42:count ───┤    { "user:42:count": 73 }
Gateway4 ──── INCR user:42:count ───┘

All 4 gateways see the same counter.
When it hits 100 → all 4 return 429.
```

`INCR` = Redis command that atomically increments a counter. Atomic = no race condition where two gateways both read 99 and both allow the request.

**One-line summary:** Distributed rate limiting requires a shared counter (Redis) — local memory per gateway gives each instance a separate bucket, letting clients bypass limits.

---

### Rate Limiting Strategies Comparison

| Strategy | How It Works | Best For |
|---|---|---|
| Fixed window | Count requests per time window (e.g. 100/min) | Simple, but allows bursts at window boundaries |
| Sliding window log | Track timestamp of each request | Accurate, but memory-intensive |
| Token bucket | Tokens refill at fixed rate, each request costs a token | Allows controlled bursts |
| Leaky bucket | Requests queue and drain at fixed rate | Smooth output rate |

---

## Code / Config Example

### Kong (declarative YAML config)

Kong = popular open-source API gateway. Configured via YAML — no code in backend services.

```yaml
# kong.yaml

services:
  - name: user-service
    url: http://user-service:3001   # upstream (internal, no TLS needed)

  - name: order-service
    url: http://order-service:3002

routes:
  - name: users-route
    service: user-service
    paths:
      - /users

  - name: orders-route
    service: order-service
    paths:
      - /orders

plugins:
  - name: jwt
    service: order-service
    config:
      claims_to_verify:
        - exp                       # verify token hasn't expired

  - name: rate-limiting
    service: order-service
    config:
      minute: 100                   # 100 requests per minute
      policy: redis                 # shared Redis counter (distributed-safe)
      redis_host: redis
      redis_port: 6379
```

JWT validation and rate limiting in config only. Backend services have zero auth/rate-limit code.

---

### Gateway Pipeline in Node.js (simplified — shows the mechanic)

```js
app.use(async (req, res, next) => {
  // Step 1: validate JWT
  const token = req.headers.authorization?.split(' ')[1]
  if (!token) return res.status(401).json({ error: 'Missing token' })

  let decoded
  try {
    decoded = jwt.verify(token, process.env.JWT_SECRET)
  } catch {
    return res.status(401).json({ error: 'Invalid token' })
  }

  // Step 2: rate limit check (Redis)
  const key = `rl:${decoded.user_id}`
  const count = await redis.incr(key)
  if (count === 1) await redis.expire(key, 60)  // set 60s TTL on first request
  if (count > 100) return res.status(429).json({ error: 'Rate limit exceeded' })

  // Step 3: inject identity, forward to upstream
  req.headers['x-user-id'] = decoded.user_id
  delete req.headers['authorization']   // strip JWT — backend doesn't need it
  next()
})
```

---

## Trade-offs & When NOT To Use

| Situation | Use a Gateway? |
|---|---|
| Monolith with one backend | No — Nginx reverse proxy (D07) is enough |
| Multiple microservices | Yes — centralizes all cross-cutting concerns |
| Single-dev side project | Probably no — over-engineering |
| Public API with third-party clients | Yes — rate limiting + API key auth are essential |
| Internal service-to-service traffic | Maybe not — service mesh (Envoy/Istio) is better |

**Single point of failure.** If gateway goes down, all traffic dies. Fix: multiple gateway instances behind a load balancer (D08).

**Adds latency.** ~1–5ms per request for the pipeline. Invisible for most apps; matters for ultra-low-latency systems.

**Can become a bottleneck.** Heavy transformations or slow plugins make the gateway the slowest thing in your stack. Keep gateway logic lightweight — push business logic to backend services.

---

## Connections to Previous Sessions

- **D07** — Nginx as reverse proxy was the first hint at this pattern. Gateway = Nginx with auth, rate limiting, and routing config baked in.
- **D08** — Load balancing across multiple gateway instances requires shared Redis for rate limiting. Sticky sessions (D08) are not a solution here.
- **D09** — JWT validation logic from D09 runs at the gateway. Auth offloading means backends never see a JWT.
- **D05** — REST APIs use Bearer tokens in `Authorization` header — the gateway reads exactly that header.

---

## Quiz

**Q1:** You have 6 microservices. Each one currently validates JWTs and checks rate limits on its own. What's the concrete problem, and how does an API Gateway fix it?

**A:** JWT validation and rate limiting code is duplicated in 6 places — 6 places to have a bug, 6 places to update when the secret rotates. The gateway centralizes both in one place; backend services just read injected headers.

---

**Q2:** A request hits your API Gateway with `Authorization: Bearer eyJ...`. The gateway validates it and it's valid. What exactly does the gateway forward to the upstream service, and what does it NOT forward?

**A:** Gateway forwards the request with `X-User-Id: 42` (injected from JWT payload). It strips the `Authorization` header — the upstream service never sees the JWT. Upstream trusts `X-User-Id` from the gateway.

---

**Q3:** You run 3 API Gateway instances. Rate limit is 60 req/min per user. You use in-memory counters. A user sends 60 requests — 20 reach each instance. What happens, and how do you fix it?

**A:** Each instance sees 20 requests — under 60 — so all pass. User effectively gets 180 req/min. Fix: shared Redis counter. All instances `INCR` the same key; when count hits 60 globally, all instances return 429.

---

## Follow-up Q&A from Session

**Q: If Nginx terminates TLS (D07), and the API Gateway also terminates TLS — aren't both running on the same request?**

A: No — TLS terminates exactly once per request path. There are three common setups:
1. **Nginx only** (small apps) — Nginx terminates TLS, forwards to one backend. No gateway.
2. **API Gateway only** (most common for microservices) — Gateway sits at the edge, handles TLS itself. No separate Nginx.
3. **LB → API Gateway** (large scale) — Load balancer or Nginx terminates TLS, passes plain HTTP to gateway, gateway handles auth/routing.

When someone says "we use an API Gateway" in a system design context, assume setup 2.

---

**Q: Why not just save rate limit counters to PostgreSQL instead of Redis?**

A: You could, but databases are slow for this. Every request would need a DB read + write on the hot path. Redis lives entirely in RAM — reads/writes take microseconds vs milliseconds for a disk-based DB. Fast enough to sit in front of every request. The "shared server" insight is the key point; Redis is just the standard choice because of the speed.
