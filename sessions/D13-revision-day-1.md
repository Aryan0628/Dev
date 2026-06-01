# D13 — Revision Day 1: D06–D12 (Core Backend Concepts)

**Date:** 2026-06-01
**Phase:** 1 — Foundations (Wrap-up)

---

## What This Session Covers

Full theory recap + quiz for D06–D12. Format per day: theory summary → quiz questions → answers.

---

## D06 — Processes vs Threads vs Async

### Theory

**The problem:** When your server waits for a DB response (10–50ms), what does the CPU do — sit idle or handle other requests?

**Process** = independent program, own memory, own FD table. ~1MB RAM, ~5ms to spawn. Fully isolated.

**Thread** = execution unit inside a process. Shares heap + globals with other threads. Own stack only (~8KB). Sockets are file descriptors stored in the process's FD table — any thread can access any socket, not just the one that opened it.

**GIL** (Global Interpreter Lock) — Python only. Only one thread runs Python bytecode at a time, even on 8 cores. Threads are concurrent (taking turns), not parallel (simultaneously). I/O releases the GIL so other threads can run. Node.js is actually single-threaded — one JS thread, no GIL needed.

**Thread states:**
- `RUNNING` → on CPU right now
- `READY` → wants to run, waiting for a free core
- `BLOCKED` → waiting for I/O or a lock — completely off CPU, OS timer does not fire on it

**Blocking I/O** — `read()` puts the thread to sleep (BLOCKED) until data arrives.
**Non-blocking I/O** — `read()` returns immediately with `EAGAIN` if no data ready. Thread keeps going.
**epoll** (Linux) / **kqueue** (macOS) — register many file descriptors, `epoll_wait()` blocks until any one has data. One thread watches thousands of sockets simultaneously.

**The three models:**

| Model | How it handles connections | Problem |
|---|---|---|
| Process per connection | Fork a new process per client | ~1000 processes = huge memory |
| Thread per connection | New thread per client | GIL limits CPU parallelism |
| Async event loop (Model 3) | One thread + epoll, all connections | CPU work blocks everything |

**`await`** = pause this function, hand the thread back to the event loop. OS watches the socket. Thread runs other coroutines. Resumes when data arrives. A CPU-bound task that never hits `await` freezes the whole event loop — no other request is handled until it finishes.

---

### Quiz

**Q1.** A Python web server uses threads and gets 1,000 concurrent requests. Each request spends 40ms waiting for a DB query. Will those threads actually run in parallel on 8 CPU cores? Why or why not?

**Q2.** You have a single-threaded async server (Node.js). A request comes in that needs to resize a 50MB image — pure CPU work, takes 2 seconds. What happens to all other incoming requests during those 2 seconds?

**Q3.** What is the difference between blocking I/O and non-blocking I/O at the OS level? What syscall does the OS provide to efficiently watch many sockets at once?

---

### Answers

**A1.** No. Python's GIL means only one thread executes Python bytecode at a time, even on 8 cores. The 1,000 threads are real OS threads but concurrent, not parallel. However, each thread releases the GIL during the 40ms DB wait — so other threads can run their Python code in that window. Still useful for I/O-bound work despite the GIL.

**A2.** All other requests freeze. The CPU work never hits an `await`, so the coroutine never yields back to the event loop. The single thread is stuck for 2 seconds — no other request is handled. Fix: offload CPU work to a thread pool or worker process, then `await` the result.

**A3.** Blocking I/O: `read()` puts the thread to sleep (BLOCKED) until data arrives — thread does nothing. Non-blocking I/O: `read()` returns immediately with `EAGAIN` if no data ready — thread continues. The syscall: `epoll` (Linux) / `kqueue` (macOS) — register many FDs, `epoll_wait()` blocks until any one has data, returns which ones fired.

---

## D07 — How a Web Server Works (Nginx Internals)

### Theory

**"Server" has 3 meanings** — a physical machine, a web server process (Nginx), or your app (Node.js). All three can live on the same machine.

**What Nginx is at OS level:** A process. At startup: master reads config → opens port 443 → forks N worker processes (N = CPU core count) → workers run epoll event loops (Model 3 from D06). Master never touches a request again. Workers never fork again after startup — that's the critical difference from Model 1 (which forks per connection).

**Master vs Worker:**
- Master: 1 process, manages workers, never handles requests
- Workers: 1 per CPU core, each runs an epoll loop, handles thousands of connections concurrently

**The two TCP connections on every proxied request:**
```
Connection A: Client ←→ Nginx     (port 443, encrypted TLS, Nginx is the server)
Connection B: Nginx  ←→ Your App  (localhost:3000, plain HTTP, Nginx is the client)
```
TLS terminates at Nginx. Your app never handles TLS. Connection B comes from a small shared pool (~32 connections) reused across all clients — not one connection per client.

**Why TCP even on the same machine:** Nginx and Node.js are separate processes. Processes can't share memory. The only way to communicate is a network socket — `127.0.0.1` (loopback, never leaves the machine).

**Port ownership:** One process calls `bind(443)` once. Thousands of clients can still connect to port 443 simultaneously — each gets its own connection socket identified by a unique 4-tuple (client IP, client port, server IP, server port).

**Source IP problem:** Your app sees `127.0.0.1` as the client IP because Connection B is opened by Nginx from localhost. Fix: `proxy_set_header X-Real-IP $remote_addr;` — injects the real client IP as a header.

---

### Quiz

**Q1.** Nginx has 4 worker processes and `worker_connections 2048`. What is the max simultaneous connections Nginx can handle? Why does Nginx use one worker per CPU core instead of one worker per connection?

**Q2.** A client's request arrives at Nginx. Nginx proxies it to your app on `localhost:3000`. Describe the two connections involved — what is Connection A, Connection B, which is encrypted, and who opened each TCP connection?

**Q3.** Your Node.js app logs every request's source IP. Without special Nginx config, every log shows `127.0.0.1`. Why? What config line fixes this and what does it do exactly?

---

### Answers

**A1.** 4 × 2048 = **8,192 simultaneous connections.** One worker per core because each worker runs an epoll event loop (Model 3) — it handles thousands of connections with one thread. More workers than cores wastes CPU on context switching with no benefit. No need for a worker per connection — epoll solves concurrency without spawning anything.

**A2.** Connection A: Client ↔ Nginx, opened by the client, port 443, **encrypted TLS.** Connection B: Nginx ↔ App, opened by Nginx (Nginx acts as TCP client), `127.0.0.1:3000`, **plain HTTP.** TLS terminates at Nginx — decrypted before being forwarded as plain HTTP on Connection B.

**A3.** The app sees `127.0.0.1` because Connection B is opened by Nginx from localhost — that's the TCP source IP of the request the app receives. Fix: `proxy_set_header X-Real-IP $remote_addr;` — injects the real client IP (source of Connection A) as an HTTP header. Your app reads `X-Real-IP` instead of the TCP source.

---

## D08 — Load Balancing

### Theory

**Purpose:** One server can't handle 50,000 req/s. You run 10 and need something to distribute traffic. The load balancer is the single entry point — if a backend dies, traffic shifts silently.

**L4 vs L7:**

| | L4 | L7 |
|---|---|---|
| Sees | IP + port + raw TCP bytes | Full HTTP (URL, headers, body) |
| Reads HTTP? | No | Yes (must terminate TLS first) |
| Speed | Faster | ~1ms overhead |
| Use case | Non-HTTP, raw TCP, strict E2E encryption | HTTP APIs, path-based routing |

**Algorithms:**
- **Round Robin** — cycles 1→2→3→1. Ignores actual load.
- **Weighted Round Robin** — `weight=3` gets 3× more requests. Use when hardware differs.
- **Least Connections** — routes to the server with fewest active connections. Best for variable-length requests (uploads, streams).
- **IP Hash** — `hash(client_IP) % N` → always same server. Problem: NAT (Network Address Translation) — offices/universities share one IP, all their traffic hits one backend.

**Sticky Sessions (Session Affinity):** LB issues a cookie (`SERVERID=s2`). All future requests from that client go to Server 2. Needed when session state is in server memory. The real fix: store sessions in Redis — any server can read them, sticky sessions become unnecessary, and you don't lose sessions when a server dies.

**L7 internals:** Terminates client TCP + TLS → parses full HTTP → routing decision → opens new TCP + TLS to chosen backend. Two separate TLS connections.

---

### Quiz

**Q1.** You have 3 backends. Server 1 is handling a 30-second video upload. A new request arrives. Which algorithm avoids Server 1, and which might hit it?

**Q2.** User logs in on Server 2 (session stored in Server 2's memory). They refresh and hit Server 3. What happens? What are the two solutions?

**Q3.** What does an L4 vs L7 load balancer see for a `GET /api/users` request?

---

### Answers

**A1.** **Least Connections** avoids Server 1 — it has the most active connections right now. **Round Robin** might hit it — it just cycles in order, ignores load.

**A2.** Session doesn't exist on Server 3 — user appears logged out. **Fix 1:** Sticky sessions (always route this client to Server 2 via cookie). **Fix 2:** Store sessions in Redis — any server looks it up. Option 2 is the correct long-term fix.

**A3.** **L4** sees: source IP, source port, destination IP, destination port — raw TCP. Cannot read the path at all. **L7** sees: full HTTP request — method `GET`, path `/api/users`, all headers, body. Had to terminate TLS first to read any of it.

---

## D09 — Auth: Sessions vs JWT vs OAuth 2.0

### Theory

**The problem:** HTTP is stateless (D02) — every request is independent, server has zero memory of previous ones. Auth is the mechanism that attaches identity to a stateless protocol.

**Sessions (stateful):**
Server stores session data in Redis. Client holds only a random meaningless ID in a cookie (`sid=a7f3b2`). Every request: server looks up the ID in Redis → finds the user.

Cookie flags that matter:
- `HttpOnly` — JavaScript on the page cannot read the cookie (blocks XSS theft)
- `Secure` — only sent over HTTPS
- `SameSite=Strict` — only sent to same origin, blocks CSRF

**JWT — JSON Web Token (stateless):**
Structure: `header.payload.signature` — three base64url-encoded parts, dot-separated.
- Header: algorithm used
- Payload: claims like `{ user_id: 42, exp: 1717000000 }`
- Signature: `HMAC_SHA256(header + "." + payload, SERVER_SECRET)`

Server verifies by recomputing the signature — no DB/Redis lookup needed. State lives on the client.

Core weakness: **can't revoke before expiry.** Banned user with a valid JWT = they're still in until it expires. A revocation blocklist fixes this but reintroduces statefulness.

The payload is **not encrypted** — only signed. Anyone can base64-decode it. Never put passwords or sensitive data in a JWT payload.

**OAuth 2.0 (delegated authorization):**
Three parties: Resource Owner (you), Client (your app), Authorization Server (Google/GitHub).

Why two-step (code → token)? The `?code=AUTH_CODE` arrives in the browser URL — visible in logs and browser history. The code alone is useless without `client_secret` (only your backend knows it). The final token exchange is server-to-server, never visible to the browser.

`state` parameter = random nonce stored in your session before redirecting to Google. Returned in the callback. If they don't match → attacker injected a fake callback → request killed. This blocks OAuth CSRF attacks.

---

### Quiz

**Q1.** HTTP is stateless. Explain in one sentence why this makes authentication non-trivial.

**Q2.** You issue a JWT. The user gets banned 5 minutes later. JWT expires in 55 minutes. What happens when they hit a protected route?

**Q3.** In OAuth's Authorization Code flow, why is there a two-step process (code first, then exchange for token) instead of redirecting back with the access token directly?

---

### Answers

**A1.** Because each request carries no memory of previous ones, the server has no way to know "this is the same person who logged in" without the client proving identity on every single request.

**A2.** They get in. The signature is valid, expiry hasn't hit — the server has no way to know they're banned without a separate blocklist. This is the core weakness of stateless tokens.

**A3.** The code arrives via browser redirect — visible in the URL, logs, browser history. The code alone is useless without `client_secret` (only your backend knows it). The token exchange is server-to-server, never exposed to the browser.

---

## D10 — API Gateway

### Theory

**Problem:** Without a gateway, every microservice re-implements JWT validation, rate limiting, TLS, and logging independently — dozens of places to have a bug, dozens of places to update when anything changes.

**API Gateway = one front door.** Cross-cutting concerns handled in one place. Request pipeline:
1. TLS termination
2. Auth check → fail = 401, stop here
3. Rate limit check → fail = 429, stop here
4. Route matching (`/users/*` → user-service)
5. Forward request (strip JWT, inject `X-User-Id: 42`)
6. Return response

**Auth offloading:** Gateway validates JWT, strips the `Authorization` header, injects `X-User-Id: 42` as a plain header. Backend services trust that header without knowing JWT exists. JWT validation lives in exactly one place.

**Token Bucket algorithm:** Bucket holds max N tokens, refills at R tokens/sec. Each request costs 1 token. Allows controlled bursts — better UX than a hard per-second cap.

**Distributed rate limiting problem:** 4 gateway instances, each with in-memory counters. User sends 100 req, load balancer spreads 25 to each. Each sees 25 — under the limit. User gets 4× their allowed rate through the system. Fix: all instances `INCR` the same key in shared Redis. When the global count hits the limit, all instances return 429.

---

### Quiz

**Q1.** You have 6 microservices. Each one currently validates JWTs and checks rate limits on its own. What's the concrete problem, and how does an API Gateway fix it?

**Q2.** A request hits your API Gateway with `Authorization: Bearer eyJ...`. The gateway validates it — it's valid. What exactly does the gateway forward to the upstream service, and what does it NOT forward?

**Q3.** You run 3 API Gateway instances. Rate limit is 60 req/min per user. You use in-memory counters. A user sends 60 requests — 20 reach each instance. What happens, and how do you fix it?

---

### Answers

**A1.** JWT validation and rate limiting code is duplicated in 6 places — 6 places to have a bug, 6 places to update when the secret rotates. The gateway centralizes both; backend services just read injected headers.

**A2.** Gateway **forwards:** the request with `X-User-Id: 42` (decoded from the JWT payload). Gateway **strips:** the `Authorization: Bearer eyJ...` header — the upstream service never sees the JWT.

**A3.** Each instance sees 20 requests — under 60 — so all pass. User effectively gets 180 req/min. Fix: shared Redis counter. All instances `INCR` the same key. When the count hits 60 globally, all instances return 429.

---

## D11 — WebSockets vs SSE vs Long Polling

### Theory

**The problem:** HTTP is request-response. The server can't push data without the client asking first. Chat messages, live prices, collaborative edits — the server needs to reach out.

**Long Polling:** Standard HTTP, but the server holds the response open until it has data (or a timeout fires). Client immediately re-requests after each response — always exactly one open connection. Problems: proxy idle timeouts (Nginx default: 65s) silently kill connections; header overhead on every reconnect cycle.

**SSE (Server-Sent Events):** One long-lived HTTP response, server keeps writing chunks, never closes the connection. Uses `Content-Type: text/event-stream`. Browser's built-in `EventSource` API handles parsing and automatic reconnection with `Last-Event-ID`. One-way: **server → client only.** Client sends data via normal HTTP requests. HTTP/2 multiplexes multiple SSE streams over one TCP connection.

**WebSocket:** Separate protocol (`ws://`, `wss://`). Starts as HTTP, upgrades mid-connection with `HTTP 101 Switching Protocols`. After upgrade, both sides exchange WebSocket frames — not HTTP. **Fully bidirectional.** Ping/pong frames for keepalive — critical because Nginx's `proxy_read_timeout` (65s default) kills idle connections silently.

| | Long Polling | SSE | WebSocket |
|---|---|---|---|
| Direction | Server→Client | Server→Client | Both |
| Protocol | HTTP | HTTP | ws:// |
| Auto-reconnect? | Manual | Built-in | Manual |
| Use case | Simple, infrequent updates | Dashboards, feeds | Chat, gaming, collaboration |

**Cross-instance WebSocket problem:** Client A on Instance 1, Client B on Instance 2. A sends a message — Instance 1 broadcasts to its local clients only — B never gets it. Fix: Redis pub/sub. Instance 1 publishes to `room:42`, all instances are subscribed, each broadcasts to its local clients in that room.

> **Real case: Slack — 2015–2016**
> Mass reconnect events (phone unlock, network switch) caused all disconnected clients to retry simultaneously with the same fixed delay — thundering herd. Fix: exponential backoff + random jitter on reconnect.

---

### Quiz

**Q1.** You're building a live dashboard showing server CPU and memory metrics, updated every second, displayed in a browser. Data flows one way (server → client). Which technique do you choose and why?

**Q2.** A WebSocket connection through Nginx drops exactly at 65 seconds of no messages. What's the cause and how do you fix it?

**Q3.** Your chat app uses WebSockets on 3 backend instances behind a round-robin load balancer. Users report messages aren't reaching everyone. What's the root cause?

---

### Answers

**A1.** **SSE.** Data is server → client only — WebSocket's bidirectionality is wasted complexity. SSE has built-in reconnect, works over plain HTTP (no protocol upgrade), and gets HTTP/2 multiplexing for free. Long polling adds one RTT (round-trip time) of extra latency per update and reopens a connection every second — wasteful.

**A2.** Nginx's default `proxy_read_timeout` is 65s. An idle WebSocket with no frames in 65s looks like a stalled request to Nginx — it kills it. Two fixes: (1) increase `proxy_read_timeout` in the Nginx location block (e.g. 3600s), (2) implement server-side WebSocket ping/pong so a frame travels before the timeout fires.

**A3.** No shared pub/sub. Client A is on Instance 1, Client B on Instance 2. Instance 1 broadcasts A's message to its locally connected clients only — B on Instance 2 never receives it. Fix: Redis pub/sub — Instance 1 publishes to the room channel, all instances receive it, each broadcasts to its local clients in that room.

---

## D12 — API Versioning

### Theory

**The problem:** You can't update all consumers the moment you update your server — they're independent systems. Versioning gives clients a runway to migrate.

**Breaking vs non-breaking changes:**
- **Non-breaking (safe):** adding a new response field, adding a new optional param, adding a new endpoint
- **Breaking (requires new version):** renaming a field, removing a field, changing a field type, changing auth mechanism

Most common mistake: engineers introduce breaking changes thinking they're "just a cleanup" without versioning — clients reading a now-missing field get silent 500s.

**The four strategies:**

| Strategy | Version lives in | Cacheable? | Testable in browser? |
|---|---|---|---|
| URL path `/v2/users/123` | URL | Yes | Yes |
| Header `Accept-Version: v2` | Request header | Only with `Vary` | No |
| Query param `?version=2` | Query string | With care | Yes |
| Media type `Accept: vnd.app.v2+json` | Accept header | Only with `Vary` | No |

URL path versioning wins in practice — visible, cacheable, loggable, easy to route at Nginx/gateway level. Used by Stripe, GitHub, Google APIs.

**`Vary` header trap:** Header versioning without `Vary: Accept-Version` in the response → CDN serves one cached response to everyone regardless of which version they requested. Silent, hard-to-debug production bug.

**Deprecation lifecycle:**
```
New version released
        |
[Both versions live]  ← Migration window (6–12 months)
        |
[Deprecated version]  ← Add Sunset + Deprecation headers here
        |
Sunset date → old version removed
```

`Sunset: <date>` and `Deprecation: true` in response headers = machine-readable warning. Monitoring tools on the client's side can detect them and alert their team. Emails and blog posts reach no one who isn't actively watching.

> **Real case: Twitter — 2013**
> Deprecated REST API v1 with a hard shutdown date and no programmatic signals — just blog posts. Thousands of third-party apps went dark overnight. Developers cite this as a cautionary tale 10 years later.

---

### Quiz

**Q1.** You add a new optional field `middle_name` to your `GET /v1/users/:id` response. A client that doesn't know about it calls the endpoint. Does this break them? Do you need a new version?

**Q2.** You're using header versioning (`Accept-Version: v2`). Your response goes through a CDN. What must you include in your response to prevent v1 clients from getting a v2 cached response?

**Q3.** You shut down v1 of your API. Three months later a large enterprise client's integration breaks — they say they never got any warning. What should you have done during the deprecation period to give them a machine-readable warning?

---

### Answers

**A1.** No, it does not break them. Adding a new field is non-breaking — well-written clients follow the robustness principle and ignore unknown fields. You do **not** need a new version.

**A2.** Include `Vary: Accept-Version` in the response. This tells the CDN to cache separately per value of the `Accept-Version` request header. Without it, one cached response gets served to all clients regardless of which version they requested.

**A3.** Add `Sunset: <date>` and `Deprecation: true` response headers to all v1 responses during the deprecation window. A monitoring tool on the client's side can detect `Sunset` programmatically and alert their team. Relying on emails and blog posts misses clients who never read them.
