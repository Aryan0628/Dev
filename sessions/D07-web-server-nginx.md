# D07 — How a Web Server Works: Nginx Internals + Reverse Proxy

**Date:** 2026-05-25

---

## One-Line Summary

Nginx is just an OS process — its master forks workers once at startup, each worker runs an epoll event loop (D06 Model 3), and when used as a reverse proxy, every client request involves exactly two TCP connections: Connection A (client→Nginx, encrypted) and Connection B (Nginx→app, plain HTTP, reused from a shared pool).

---

## Word Overload Warning — "server" has three meanings

| Context | "Server" means |
|---|---|
| Infrastructure | A physical or virtual machine (the computer) |
| Web server | A software process that speaks HTTP and listens on a port (Nginx, Apache) |
| Application server | Your own code — Node.js, Python Flask, Go, etc. |

---

## What Nginx Is, at the OS Level

Nginx is a process. When it starts:

```
1. Nginx master process starts
2. Reads config file (/etc/nginx/nginx.conf)
3. Opens a listening socket on port 443 (or 80)
4. Forks N worker processes (N = CPU cores, typically)
5. Master goes to sleep — workers handle requests
```

Master process never touches a client request. Workers run epoll event loops (D06 Model 3).

---

## Master vs Worker

| | Master Process | Worker Process |
|---|---|---|
| Count | 1 | 1 per CPU core |
| Handles requests? | No | Yes |
| Shares memory with workers? | No (separate processes) | No (fully isolated) |
| What it does | Config, fork/kill workers, signals | Runs epoll event loop |

**Workers are OS processes, not threads.** Isolated in memory. One bug in Worker 1 cannot corrupt Worker 2.

**Each worker is not "process per connection" either.** Each worker is a single process running a single epoll event loop — handles thousands of connections concurrently with one thread. Same as D06 Model 3.

### How the listening socket is shared

Master opens listening socket (fd 3, port 443) before forking. Fork copies FD table — all workers get fd 3. All workers call `accept()` on the same fd 3. OS only wakes one worker per incoming connection (via `EPOLLEXCLUSIVE`).

**One-line:** Listening socket is shared. Connection sockets are not — each accepted connection belongs to exactly one worker.

---

## Nginx's Concurrency Model — The Hybrid

Nginx is NOT a new concurrency model. It combines two things from D06:

```
Outer layer: Multiple worker PROCESSES  → CPU parallelism + crash isolation
Inner layer: Each worker runs Model 3   → async event loop + epoll for I/O concurrency
```

| | D06 Model 1 | D06 Model 3 | Nginx |
|---|---|---|---|
| Processes created | 1 per connection (on demand) | 1 total | 1 per CPU core (at startup, fixed) |
| How connections handled | 1 blocking thread per process | epoll, 1 thread, all connections | epoll, 1 thread per worker, all connections |
| 1000 clients cost | ~1000 processes | 1 process | 4 processes (fixed) |

**Critical:** Nginx forks workers ONCE at startup. Never again. A new connection does NOT trigger a fork — it gets added to a worker's epoll list. This is what makes Nginx different from Model 1.

```
Model 1: Client connects → FORK new process ← per connection
Nginx:   Client connects → add to epoll    ← no fork, ever
```

---

## Two Modes: Static vs Reverse Proxy

```
Mode 1 — Static file server:
  Client → Nginx → reads from disk → Client
  (your app is never involved)

Mode 2 — Reverse proxy:
  Client → Nginx → Your App → Nginx → Client
  (almost all production setups)
```

---

## Forward Proxy vs Reverse Proxy

| | Forward Proxy | Reverse Proxy |
|---|---|---|
| Acts on behalf of | The **client** | The **server** |
| Who doesn't know the real party | The destination server | The client |
| Examples | VPN, corporate firewall | Nginx, load balancer, CDN |
| Client configures it? | Yes | No — client has no idea it exists |

**Forward proxy:** You hire someone to go to the store for you. The store deals with your person, not you.

**Reverse proxy:** A company receptionist. You talk to the receptionist, they route you internally. You have no idea what's behind the desk.

**Why "reverse"?** The proxy is on the server's side (reverse direction vs forward proxy on client's side).

---

## The Two TCP Connections — Most Important Mechanic

Every proxied request involves two separate TCP connections:

```
Connection A:
  Client ←─── TCP ───→ Nginx
  Port 443, encrypted with TLS

Connection B:
  Nginx ←─── TCP ───→ Your App
  localhost:3000, plain HTTP (no TLS)
```

Worker FD table:
```
fd 3  → listening socket (port 443)
fd 47 → Connection A (client ↔ Nginx)
fd 52 → Connection B (Nginx ↔ app)
```

Nginx is simultaneously:
- A **server** for Connection A (client connects to Nginx)
- A **client** for Connection B (Nginx connects to your app)

### Why does fd52 exist at all?

Nginx and Node.js are two separate processes. Separate processes cannot share memory. The only way two processes communicate — even on the same machine — is over a network connection. So Nginx must open its own TCP connection to Node.js, exactly like a browser opens a connection to a server. fd52 is that connection.

```
Previous sessions (D01-D06): one connection, one flow
  Client → [TCP fd47] → Server (your app)

D07 with Nginx: two connections, middleman
  Client → [TCP fd47] → Nginx → [TCP fd52] → Node.js
```

fd52 exists because the middleman needs its own pipe to talk to the backend.

---

## Nginx and Node.js — Same Machine, Same OS

They are NOT on different operating systems. Both are processes on the same machine, sharing the same kernel.

```
One machine, one OS:
├── Nginx worker process (PID 101) → port 443 (internet-facing)
├── Node.js process (PID 200)      → port 3000 (hidden, internal only)
└── Kernel manages both processes, both TCP connections
```

They communicate via `localhost` (127.0.0.1) — a loopback interface that never leaves the machine. No physical network involved. Both processes use epoll + async event loop (D06 Model 3), just watching different FDs for different purposes.

| | Nginx epoll | Node.js epoll |
|---|---|---|
| Watches | Client sockets + upstream sockets | Incoming request sockets + DB/Redis sockets |
| Wakes up when | Client sends data OR Node.js responds | Nginx sends request OR DB responds |
| What it does on wake | Route/proxy the request | Run your business logic |

---

## Port Ownership — One Process, One Port

Only one process can call `bind()` on a port. `bind()` is the syscall that claims a port at startup.

```
Nginx      → bind(443) ✓
Node.js    → bind(443) ✗ ERROR: address already in use
```

This does NOT mean only one connection can exist on that port. Many clients can all connect to port 443 — each gets their own connection socket (FD), identified by the 4-tuple (client IP, client port, server IP, server port). The port binding is just the initial claim.

```
Port 443 owned by Nginx (one bind):
  Client 1: (1.1.1.1, 54321, server, 443) → fd47
  Client 2: (2.2.2.2, 54322, server, 443) → fd52
  Client 3: (3.3.3.3, 54323, server, 443) → fd63
  All three on same port 443 — no conflict
```

This is why running 4 Node.js instances directly on port 443 fails — all 4 try to `bind(443)`, only the first succeeds. Solution: Nginx binds 443, each Node.js gets its own port (3000, 3001, 3002, 3003).

---

## CPU Cores — Not Owned by Processes

"One worker per CPU core" does NOT mean workers monopolize cores. The OS scheduler shares cores across all processes.

Nginx workers are BLOCKED (in `epoll_wait()`) most of the time — not using CPU. BLOCKED = timer off, not on CPU (D06). When workers are blocked, Node.js gets the cores freely.

```
Core 1: Nginx Worker 1 (BLOCKED 99% of time) ←→ Node.js (runs here)
Core 2: Nginx Worker 2 (BLOCKED 99% of time) ←→ Node.js (runs here)
```

"One worker per core" is a performance hint: when a worker DOES wake up and need CPU, it gets a core without fighting another Nginx worker for it (no context switching between workers).

---

## TLS Termination

"Termination" = where TLS ends. TLS terminates at Nginx.

```
[Client]
   │  HTTPS (encrypted)
   ↓
[Nginx] ← TLS terminates here. Nginx decrypts.
   │  HTTP (plain text)
   ↓
[Your App]
```

Why terminate at Nginx:
1. App code has no TLS logic needed
2. One certificate, one place to manage
3. New app instances get HTTPS automatically
4. Nginx is faster at TLS than application frameworks

The `.crt` and `.key` files (D04) go in Nginx config. Nginx reads them at startup.

---

## SNI — One IP, Multiple Domains

**SNI** = Server Name Indication. TLS ClientHello extension where client sends target hostname before encryption, so Nginx picks the right certificate.

```
TLS ClientHello:
  SNI: "api.example.com"   ← sent before TLS is established
  ALPN: ["h2", "http/1.1"]

Nginx reads SNI → picks matching server block → uses its certificate
```

Without SNI: one IP = one TLS certificate.
With SNI: one IP can host hundreds of HTTPS domains.

---

## OS vs App Boundary

```
┌──────────────────── Kernel (OS) ─────────────────────┐
│  Receives SYN → TCP handshake → creates fd 47        │
│  TLS: Nginx calls recv/send; kernel moves bytes      │
│  Manages both Connection A and Connection B TCP      │
└───────────────────────────────────────────────────────┘
            ↕ syscalls (read, write, accept, connect)
┌──────────────────── Nginx Worker ────────────────────┐
│  epoll_wait() — watching fd 47, fd 52, thousands    │
│  fd 47 has data → read() → TLS bytes                │
│  TLS library decrypts → plain HTTP                  │
│  Parse method, URL, headers                         │
│  Match URL against location blocks                  │
│  /api/* matches → proxy_pass                        │
│  Grab fd 52 from keepalive pool (or open new TCP)   │
│  Write HTTP request on fd 52                        │
│  Register fd 52 with epoll → go handle other FDs   │
│  fd 52 has response → read() → TLS-encrypt         │
│  Write encrypted bytes to fd 47 → client receives  │
└───────────────────────────────────────────────────────┘
```

When Nginx is waiting for your app to respond (fd 52), it does NOT block. It registers fd 52 with epoll and handles other connections — same event loop from D06.

---

## Connection Pooling — The Key Nginx Advantage

### Connection A (client → Nginx): one socket per client

1000 clients = 1000 Connection A sockets. Each client has their own. This is the same as any Model 3 server.

### Connection B (Nginx → app): shared pool, reused across all clients

```
Request from Client A → Nginx grabs fd52 from pool → sends to Node.js → done → fd52 back to pool
Request from Client B → Nginx grabs fd52 from pool → sends to Node.js → done → fd52 back to pool
Request from Client C → Nginx grabs fd52 from pool → sends to Node.js → done → fd52 back to pool
```

Same fd52 used by all three clients. The pool is shared — not assigned to any specific client.

**Why is this safe?** The connection to Node.js carries no client identity. It's just a pipe for HTTP requests. Node.js doesn't know fd52 was used by Client A before — it just sees a new HTTP request.

**One fd at a time rule:** fd52 can only carry one request at a time. While in use for Client A, it's not in the pool. Returns to pool only after Client A's response arrives.

```
Pool: [fd52, fd71, fd89]

Client A → grabs fd52 → pool: [fd71, fd89]
Client B → grabs fd71 → pool: [fd89]
Client C → grabs fd89 → pool: []
Client D → pool empty → opens new fd94
Client A done → fd52 returns → pool: [fd52]
```

`keepalive 32` = keep 32 idle connections per worker. 4 workers × 32 = 128 pooled connections max.

**Result:**
```
1000 clients hitting Nginx
→ 1000 Connection A sockets (one per client)
→ only ~32 Connection B sockets (pool, shared, reused)
Node.js sees 32 open connections, not 1000
```

Node.js is protected from connection flood. This is possible because Nginx controls Connection B entirely — clients can't control it.

---

## Why Nginx at All — If Node.js Already Does Model 3

Node.js already handles thousands of connections with one event loop. So why add Nginx?

| Problem | Without Nginx | With Nginx |
|---|---|---|
| Node.js uses 1 CPU core | 3 cores idle on 4-core machine | Nginx load balances across 4 Node.js instances, all cores used |
| Only 1 process can bind port 443 | 1 Node.js instance handles all traffic | Nginx owns 443, distributes to 4 instances on 3000–3003 |
| TLS management | Every app instance handles its own TLS | Nginx handles TLS once, apps get plain HTTP |
| Static files | App reads file from disk and sends it (slow) | Nginx serves directly from disk, app never involved |
| Zero-downtime deploys | Restart Node.js = ~2 sec downtime | Nginx stays up, routes to other instances during restart |
| Node.js connection flood | 1000 clients = 1000 raw connections to app | 1000 clients = ~32 pooled connections to app |

---

## Full Request Walk-Through

```
User: https://api.example.com/users/42

DNS: api.example.com → 1.2.3.4                          [D01]

TCP handshake to 1.2.3.4:443                             [D01]
  SYN → SYN-ACK → ACK
  OS creates fd 47

Nginx worker accepts from epoll:                         [D06]
  epoll_wait woke on fd 3 → accept(fd3) → fd 47

TLS handshake on fd 47:                                  [D04]
  ClientHello: SNI=api.example.com, ALPN=["h2"]
  Nginx sends certificate, ECDHE key exchange
  TLS established, ALPN picks h2

HTTP/2 request on fd 47:                                 [D03]
  Nginx decrypts → "GET /users/42 HTTP/2"

Location matching → proxy_pass to my_app                 [D07]

Nginx grabs fd 52 from keepalive pool                    [D03, D07]
  (or opens new TCP to 127.0.0.1:3000)
  Writes HTTP/1.1 request on fd 52:
    GET /users/42 HTTP/1.1
    Host: api.example.com
    X-Real-IP: 203.0.113.45

Nginx registers fd 52 with epoll → handles others        [D06]

App responds on fd 52:                                   [D05]
  HTTP 200 OK, JSON body

Nginx TLS-encrypts, writes to fd 47                      [D04, D07]

fd 47 stays open (HTTP/2)                                [D03]
fd 52 returns to pool — available for any client        [D07]
```

---

## Annotated Config

```nginx
worker_processes auto;
# One worker per CPU core. More than cores = context switch overhead (D06).

events {
    worker_connections 1024;
    # Max simultaneous connections per worker.
    # 4 workers × 1024 = 4096 total.
}

http {
    upstream my_app {
        # "upstream" = Nginx's name for your backend app server(s)
        server 127.0.0.1:3000;
        keepalive 32;
        # Keep 32 idle TCP connections to app per worker (D03 keep-alive)
        # These are shared across all clients — not per-client
    }

    server {
        listen 443 ssl;
        server_name api.example.com;

        ssl_certificate     /etc/ssl/certs/api.example.com.crt;
        ssl_certificate_key /etc/ssl/private/api.example.com.key;
        # Certificate + private key from D04. Used in every TLS handshake.

        location /static/ {
            root /var/www;
            # Served directly from disk. Connection B never opened.
        }

        location / {
            proxy_pass http://my_app;

            proxy_http_version 1.1;
            # HTTP/1.1 required on Connection B for keepalive to work

            proxy_set_header Connection "";
            # Remove Connection header so keep-alive works

            proxy_set_header Host $host;
            # Pass original Host header. Without this: app sees 127.0.0.1.

            proxy_set_header X-Real-IP $remote_addr;
            # $remote_addr = real client IP (source of Connection A)
            # Without this: app sees 127.0.0.1 for every request
            # because Connection B's source IP is always Nginx (localhost)
        }
    }

    server {
        listen 80;
        return 301 https://$host$request_uri;
        # HTTP → HTTPS redirect (301 = Moved Permanently, from D05)
    }
}
```

---

## Nginx vs Apache

| | Nginx | Apache |
|---|---|---|
| Concurrency model | Async event loop per worker (D06 Model 3) | Thread per connection (D06 Model 2) |
| Memory at 10k connections | Low (event loop) | High (10k threads × ~8KB) |
| Market share | Dominant for high-traffic | Common in shared hosting |

Apache's thread-per-connection model hits the "C10K problem" (10,000 concurrent connections) — Nginx was built specifically to solve it with the event loop model.

---

## Trade-offs & When NOT to Use Nginx

| Situation | Recommendation |
|---|---|
| Local development | Don't bother — run app directly |
| Complex routing / JWT validation | API Gateway (D10) is cleaner |
| WebSockets (D11) | Nginx supports it but needs `Upgrade` header config |
| gRPC (HTTP/2 streaming) | Use `grpc_pass` not `proxy_pass` |
| Business logic / request transformation | Put it in your app, not Nginx config |

---

## Quiz Questions + Answers

**Q1.** Nginx has 4 worker processes and `worker_connections 2048`. What is the max simultaneous connections Nginx can handle? Why does Nginx use one worker per CPU core instead of one per connection?

**A1.** 4 × 2048 = 8,192 simultaneous connections. One worker per core because each worker is a process running an epoll event loop (D06 Model 3) — it handles thousands of connections with one thread. More processes than cores would cause context switching overhead with no benefit. No need for process-per-connection — epoll solves concurrency.

---

**Q2.** A client's request arrives at Nginx. Nginx proxies it to your app on `localhost:3000`. Describe the exact FDs involved — what is Connection A, Connection B, which is encrypted, and who opened each TCP connection?

**A2.** Connection A: client ↔ Nginx, opened by the client, port 443, encrypted with TLS, Nginx has fd 47. Connection B: Nginx ↔ app, opened by Nginx (Nginx acts as TCP client), connects to 127.0.0.1:3000, plain HTTP, Nginx has fd 52. TLS terminates at Nginx — Connection A is decrypted before being re-sent as plain HTTP on Connection B.

---

**Q3.** Your Node.js app logs every request's source IP. Without special Nginx config, every log shows `127.0.0.1`. Why? What config line fixes this and what does it do exactly?

**A3.** The app sees `127.0.0.1` because the source IP of Connection B is Nginx itself — Nginx opened that TCP connection from localhost. Fix: `proxy_set_header X-Real-IP $remote_addr;` — adds an `X-Real-IP` header containing the source IP of Connection A (the real client IP). Your app reads this header instead of the TCP source IP.

---

## Follow-up Clarifications From Session

**Q: Is Nginx using a new concurrency model, different from D06's three?**
No. Nginx is a hybrid: worker processes (OS-level isolation) + Model 3 inside each worker (epoll event loop). These operate at different levels. Process = the container. Model 3 = what runs inside it. Not mutually exclusive.

**Q: Nginx worker forks multiple processes per request like Model 1?**
No — this is the critical difference. Model 1 forks per connection (on demand, unbounded). Nginx forks workers exactly once at startup (fixed = CPU core count). After startup, no more forking ever happens. Connections get added to a worker's epoll list, not a new process.

**Q: Are Nginx and Node.js on different OS/machines?**
No — same machine, same OS, same kernel. They are two separate processes communicating over localhost (127.0.0.1). The loopback interface never leaves the machine. Port 443 faces the internet; port 3000 is internal only.

**Q: Does "one process per port" mean only one connection per port?**
No. One process does the initial `bind(443)` — that's the only restriction. After binding, thousands of clients can connect to that port. Each gets their own connection socket (FD), distinguished by the 4-tuple (client IP, client port, server IP, server port). Port 443 can have 10,000 active connections simultaneously.

**Q: Can Client B reuse a pooled connection that Client A previously used?**
Yes — the Connection B pool is shared across all clients. Any request from any client can grab any idle connection in the pool. Safe because the connection to Node.js carries no client identity — it's just an HTTP pipe. The only rule: one fd handles one request at a time; it returns to the pool after the response arrives.

**Q: What is Model 3's disadvantage — the 1000 sockets?**
No — every model opens one socket per client, that's unavoidable TCP. Model 3's disadvantage is CPU-heavy work: one task that never hits `await` hogs the single thread, freezing all other clients. Sockets are not the problem; the single thread with no CPU parallelism is.
