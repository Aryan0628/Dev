# D02 — Client-Server Architecture + Request-Response Model

**Date completed:** 2026-05-20
**Time:** ~1 hour + deep Q&A on socket internals and concurrency

---

## The Core Idea

```
CLIENT  ───── request ─────▶  SERVER
        ◀──── response ─────

Client = any program that initiates a request
Server = any program that listens and responds
```

**Client/server is a ROLE, not a machine.** Your payment service is a server to the browser, but a client to Stripe.

Restaurant analogy:
- Customer = client
- Waiter = server
- Kitchen = your business logic / DB
- Menu = the API contract

---

## The Request-Response Cycle

```
CLIENT                                  SERVER
  │                                       │
  │ POST /orders HTTP/1.1                 │
  │ Host: api.shop.com                    │
  │ Authorization: Bearer eyJhb...        │
  │ Content-Type: application/json        │
  │                                       │
  │ {"item_id": 42, "quantity": 2}        │
  │ ─────────────────────────────────────▶│
  │                          1. Read      │
  │                          2. Auth      │
  │                          3. DB query  │
  │                          4. Respond   │
  │                                       │
  │ HTTP/1.1 201 Created                  │
  │ Location: /orders/8801                │
  │ {"order_id": 8801}                    │
  │ ◀─────────────────────────────────────│
```

**Request anatomy:** method + path + headers + body
**Response anatomy:** status code + headers + body

---

## Stateless vs Stateful

**HTTP is stateless** — each request is independent. The server doesn't remember previous requests.

```
Request 1:  GET /profile  (server: who is this? no idea)
Request 2:  POST /orders   (server: still no idea)
```

Every request must carry its own identity (auth token, session cookie).

**Why stateless wins at scale:**
```
Stateful:   user logged in on Server 1 → must always hit Server 1
            (any of 50 servers can't help — only Server 1 has the session)
Stateless:  any server can handle any request → horizontal scaling works
```

---

## Sockets — The Foundation

A **socket** is the OS-provided endpoint for network communication. In Unix it's just a file descriptor — you read/write it like a file.

### Two Kinds of Sockets — Critical Distinction

```
1. LISTENING SOCKET  (one per port)
   - Bound to 0.0.0.0:8000
   - NEVER carries data
   - Only job: birth new sockets when clients connect

2. CONNECTION SOCKET (one per client)
   - Created when a client connects
   - This is the actual TCP connection
   - Has its own send/receive buffers
```

```
            Listening Socket (0.0.0.0:8000)
            Job: birth new sockets
                    │
   ┌────────────────┼────────────────┐
   │                │                │
Client A        Client B        Client C
   │                │                │
   ▼                ▼                ▼
 conn1           conn2            conn3
(real TCP)     (real TCP)       (real TCP)
```

If you have 1000 clients connected → server has **1001 sockets** open.

---

## How Many Sockets Share One Port

TCP identifies connections by a **4-tuple**, not just port:

```
(source_IP, source_port, dest_IP, dest_port)

Client A:  (5.5.5.5, 51234, 1.2.3.4, 8000)
Client B:  (6.6.6.6, 49102, 1.2.3.4, 8000)
Client C:  (7.7.7.7, 60001, 1.2.3.4, 8000)
                                  ↑
                          dest is always same
                          source distinguishes them
```

When a packet arrives, OS looks up the 4-tuple → routes to the correct socket. Same client can even open multiple connections to the same port using different source ports.

---

## The 10 Steps Inside a Server

```
1. CREATE   socket(AF_INET, SOCK_STREAM)        # ask OS for a socket
2. BIND     s.bind(('0.0.0.0', 8000))           # claim a port
3. LISTEN   s.listen(128)                       # backlog queue size
4. ACCEPT   conn = s.accept()                   # blocks; returns new socket
5. READ     raw = conn.recv(4096)               # raw HTTP bytes
6. PARSE    method, path, headers, body         # framework parses HTTP
7. ROUTE    handler = routes[(method, path)]    # dict lookup
8. RUN      result = handler(request)           # YOUR code
9. SERIALIZE  bytes = build_http_response(result)
10. WRITE   conn.sendall(bytes); conn.close()
```

**Steps 1–3, 5–6, 9 are what FastAPI/Express/Django hide from you.**

### Backlog Queue

```
Clients connect faster than accept() consumes them:
  OS queues them → [A, B, C, D, ...]
  Queue fills (size 128) → new clients get "connection refused"
```

Why this matters in prod: slow accept loop → backlog overflow → users see errors before your code even runs.

### What 0.0.0.0 Means

Server has multiple network interfaces:
```
127.0.0.1   → loopback (only this machine)
192.168.1.5 → LAN
0.0.0.0     → "any interface" (production default)
```

### What `accept()` Actually Returns

The TCP handshake is done by the **OS kernel**, not your code. By the time `accept()` returns, the connection is already ESTABLISHED. `accept()` just hands you a Python object wrapping the kernel's connection.

---

## HTTP Is Just Text

What arrives on the socket:
```
b'GET /users/42 HTTP/1.1\r\nHost: api.shop.com\r\n\r\n'
```

Readable:
```
GET /users/42 HTTP/1.1\r\n
Host: api.shop.com\r\n
\r\n            ← empty line = end of headers
(body if any)
```

`\r\n` separates lines. Empty line marks end of headers. **HTTP is a text protocol** — no magic.

---

## Routing Is Just a Dictionary

```python
routes = {
    ("GET",  "/users/{id}"): get_user_handler,
    ("POST", "/orders"):     create_order_handler,
}

@app.get("/users/{user_id}")
def get_user(user_id: int):
    ...
# ↑ This decorator does:  routes[("GET", "/users/{user_id}")] = get_user
```

That's it. A router = pattern-matched dict lookup.

---

## Same Client, Two Requests — Sequential or Parallel?

Depends on the setup. **This is why HTTP/2 was invented.**

| Setup | Same TCP? | Parallel? |
|---|---|---|
| 2 TCP connections | No | **Yes — parallel** |
| 1 TCP, HTTP/1.0 | No (closes each time) | Sequential |
| 1 TCP, HTTP/1.1 keep-alive | Yes | **Sequential** (head-of-line blocking) |
| 1 TCP, HTTP/1.1 pipelining | Yes | Broken in practice |
| 1 TCP, HTTP/2 | Yes | **Yes — multiplexing** |

### Why HTTP/1.1 on one connection is sequential

A TCP connection is just a byte stream. There's no way to mark "this byte belongs to request 1 vs request 2." So the client must wait for response 1 before sending request 2.

```
Request 1: slow (500ms)
Request 2: fast (5ms)

HTTP/1.1 sequential:
  R1 → wait 500ms → resp1 → R2 → 5ms → resp2
  Total: 505ms     ← R2 stuck behind R1 = head-of-line blocking
```

### Why browsers open ~6 parallel TCP connections

To work around HTTP/1.1's sequential nature, browsers open multiple TCP connections per origin. 6 is the typical cap — more than that is wasteful (each needs TCP+TLS handshake = ~600ms).

### HTTP/2 multiplexing (preview for D03)

One TCP connection. Every chunk gets a **stream ID**:
```
[Stream 1, HEADERS, "GET /A"]
[Stream 2, HEADERS, "GET /B"]
[Stream 2, DATA, "response B"]   ← B comes back first if it's faster
[Stream 1, DATA, "response A"]
```

True parallel requests on one TCP connection. No head-of-line blocking at the HTTP layer.

---

## Concurrency Models — How One Server Handles Many Clients

Sockets existing in parallel ≠ being processed in parallel. You need a concurrency strategy.

### 1. Multi-Process (fork per connection)
```
Apache prefork style
Each connection → spawn new process
Cost: ~10–50 MB RAM per process → 1000 clients = 10–50 GB
True parallelism across CPU cores
```

### 2. Multi-Threaded
```
Thread pool, one thread per connection
Cost: ~1–2 MB per thread → better than processes
Threads share memory → need locks → bug-prone
Python GIL: only one thread runs Python at a time
  (still useful for I/O-bound work — DB waits release the GIL)
```

### 3. Async / Event Loop  ← modern default
```
Node.js, Nginx, Python asyncio, Go (kind of)
ONE thread handles 10,000+ connections

Magic tool: epoll (Linux) / kqueue (Mac)
  OS call: "Here are 10k sockets — which have data ready right now?"
  → returns only ready ones → loop processes them

Cost: ~10 KB per connection
Catch: any CPU-heavy work blocks the whole loop
       → great for I/O-bound, bad for CPU-bound
```

```
Time →
T=0ms:   conn1: read req
T=1ms:   conn1: send DB query → wait
T=1ms:   conn2: read req
T=2ms:   conn2: send DB query → wait
T=10ms:  DB responds for conn1 → send response
T=11ms:  DB responds for conn2 → send response

CPU never idle. When one connection waits, loop switches to another.
```

---

## Sync vs Async at the API Level

**Synchronous (most web requests):**
```
Client sends → WAITS → response → continue
```
Fine when you need the answer to proceed.

**Asynchronous (fire-and-forget):**
```
Client sends → server: "202 Accepted, here's a job ID"
              → server does the work later
              → client polls or gets a webhook when done
```
Use for: video encoding, report generation, anything > 30s. `202 Accepted` is the canonical async response.

---

## A Real Working Server in 20 Lines (No Framework)

```python
import socket

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(('0.0.0.0', 8000))
s.listen(128)

while True:
    conn, addr = s.accept()
    raw = conn.recv(4096)
    print(raw.decode())

    body = '{"message": "Hello from raw socket!"}'
    response = (
        "HTTP/1.1 200 OK\r\n"
        "Content-Type: application/json\r\n"
        f"Content-Length: {len(body)}\r\n"
        "\r\n"
        f"{body}"
    )
    conn.sendall(response.encode())
    conn.close()
```

Run it. `curl http://localhost:8000`. Watch the raw HTTP bytes arrive. **This IS a web server.** FastAPI/Express are just fancier versions of this.

---

## When NOT To Use Request-Response

| Situation | Problem | Better |
|---|---|---|
| Long jobs (video encode, reports) | Client times out | Async + polling/webhooks |
| Real-time updates (chat, live scores) | Polling is wasteful | WebSockets (D10) |
| Server-initiated events | Request-response is client-initiated | SSE / WebSockets |
| Millions of msgs/sec | HTTP overhead too high | Message queue (D24) |

---

## Production Gotchas

| Situation | What goes wrong |
|---|---|
| Listening on `127.0.0.1` only | Not reachable from outside the machine |
| Port already in use | Another process owns it — only one allowed |
| Backlog too small + slow accept | New connections refused before code runs |
| Server keeps state in memory | Breaks behind load balancer (sticky sessions or move to Redis) |
| Async handler does CPU-heavy work | Whole event loop freezes — all clients slow |
| HTTP/1.1 only, many small requests | Sequential per connection — slow page loads |

---

## Key Terms Glossary

| Term | Meaning |
|---|---|
| Client | Program that initiates a request |
| Server | Program that listens for and handles requests |
| Socket | OS endpoint for network communication (a file descriptor) |
| Listening socket | Bound to a port; births new sockets for each client |
| Connection socket | The actual TCP connection to one specific client |
| 4-tuple | (src_IP, src_port, dst_IP, dst_port) — uniquely identifies a TCP connection |
| Backlog | OS queue of pending connections waiting for accept() |
| Bind | Attach a socket to a specific IP:port |
| Stateless | Server keeps no memory of past requests; each request is independent |
| Keep-alive | Reuse one TCP connection for multiple HTTP requests |
| Head-of-line blocking | A slow earlier request blocks later ones on the same connection |
| Multiplexing | Many parallel streams on one TCP connection (HTTP/2) |
| epoll / kqueue | OS API: "tell me which of these sockets are ready right now" |
| Event loop | Single-threaded loop that handles many connections by switching when one is waiting |
| 202 Accepted | HTTP status meaning "received, processing async" |
| Routing | Mapping (method, path) → handler function |

---

## Quiz (open — to answer next session)

1. **Your Node.js server handles 500 concurrent users fine. You rewrite it in Java with one thread per request, max pool 100. What happens to the 101st request?**

2. **Server returns `202 Accepted` with `{"job_id": "abc123"}`. What does this tell you about how the server processed the request?**

3. **3 servers behind a load balancer. User logs in on Server 1 (session in memory). Next request goes to Server 2. What happens and how do you fix it?**
