# D08 — Load Balancing: L4 vs L7, Algorithms, Sticky Sessions

---

## Why It Matters

One server can't handle 50,000 requests/second. You spin up 10. Now you need something to distribute traffic across them — that's the load balancer. It's the single entry point for all traffic, and it decides which backend handles each request. If a backend dies, traffic shifts silently. Users notice nothing.

---

## The Big Picture

```
Client
  |
  v
[Load Balancer]          ← single entry point
  |         |
  v         v
[Server 1] [Server 2]   ← your actual app servers (upstream pool / server pool)
```

---

## L4 vs L7 — The Core Distinction

OSI model (Open Systems Interconnection) — 7-layer framework for network communication:
- L3 = IP (addresses)
- L4 = TCP/UDP (connections)
- L7 = HTTP (application requests)

| | L4 Load Balancer | L7 Load Balancer |
|---|---|---|
| Sees | IP + port + raw TCP bytes | Full HTTP request (URL, headers, body) |
| Decisions based on | Network info only | Application content |
| Reads HTTP? | No | Yes |
| Terminates TLS? | Mostly no | Yes — must, to read HTTP |
| Speed | Faster | Slower (~1ms overhead) |
| Use case | Non-HTTP, raw TCP, strict E2E encryption | HTTP APIs, path-based routing |
| Example | AWS NLB, HAProxy L4 mode | Nginx, AWS ALB, HAProxy L7 mode |

> NLB = Network Load Balancer. ALB = Application Load Balancer. E2E = End-to-End.

**One-line summary:** L4 sees TCP packets; L7 sees HTTP requests. L7 can route by content; L4 can only route by IP/port.

---

## L4 Internals

Two forwarding modes:
- **NAT mode** (Network Address Translation): rewrites destination IP in the TCP packet → backend. Client thinks it's still talking to the LB.
- **Proxy mode**: LB opens a second TCP connection to the backend and pipes bytes through.

Never decodes HTTP. Just moves TCP packets.

---

## L7 Internals

1. Terminates the client's TCP connection
2. Terminates TLS (from D04 — full TLS handshake with client)
3. Parses the full HTTP request
4. Makes routing decision (by URL, header, cookie, etc.)
5. Opens its own TCP + TLS connection to the chosen backend
6. Forwards the request

```
Client  ←TLS-A→  [L7 LB]  ←TLS-B→  [Backend]
```
Two separate TLS connections. The LB is a full participant in both.

---

## Load Balancing Algorithms

### Round Robin
Cycle through servers in order: 1 → 2 → 3 → 1 → ...
- Problem: doesn't consider actual load — a slow server still gets equal traffic.

### Weighted Round Robin
Assign weights. `weight=3` gets 3× more requests than `weight=1`.
- Use when servers have different hardware capacities.

### Least Connections (Least Conn)
Send to the server with fewest **active connections** right now.
- Smarter than Round Robin for variable-duration requests (uploads, streams).
- Most commonly recommended for general HTTP use.

### IP Hash
`hash(client_IP) % N` → always same server for same IP.
- Problem: clients behind NAT (shared IP in offices/universities) all go to one server.

### Random
Pick randomly. Statistically approaches Round Robin at scale.

---

## Sticky Sessions (Session Affinity)

Not an algorithm — a **feature** that overrides the algorithm to pin a client to the same backend.

The LB issues a cookie (e.g., `SERVERID=s2`) on first request. All subsequent requests from that client go to Server 2.

```
Request 1 → LB picks Server 2 → sets cookie SERVERID=s2
Request 2 → LB reads cookie   → routes to Server 2
Request 3 → LB reads cookie   → routes to Server 2
```

**Why you'd want it:** backend stores session state in memory (not shared DB/Redis). Different server = no session = user logged out.

**The real fix:** store sessions in Redis (shared store). Any server can read it. No sticky sessions needed.

**Danger of sticky sessions:** if Server 2 dies, everyone stuck to it loses their session anyway.

---

## Nginx Config Examples

```nginx
# Least connections across 3 backends
upstream my_app {
    least_conn;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}

server {
    listen 80;
    location / {
        proxy_pass http://my_app;
    }
}
```

```nginx
# Weighted round robin
upstream my_app {
    server 10.0.0.1:3000 weight=3;
    server 10.0.0.2:3000 weight=1;
}
```

```nginx
# IP hash (sticky by client IP)
upstream my_app {
    ip_hash;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}
```

```nginx
# Path-based routing — L7 routing by URL
upstream api_servers  { server 10.0.0.1:3000; server 10.0.0.2:3000; }
upstream image_servers { server 10.0.0.5:8080; }

server {
    listen 80;
    location /api/    { proxy_pass http://api_servers; }
    location /images/ { proxy_pass http://image_servers; }
}
```

---

## Trade-offs & When NOT to Use

| Situation | Recommendation |
|---|---|
| Single server, low traffic | Skip the LB — adds latency + complexity |
| Stateful servers | Sticky sessions short-term; Redis session store long-term |
| Route by URL path or headers | L7 (Nginx, ALB) |
| Raw TCP / non-HTTP | L4 (AWS NLB) |
| Strict end-to-end encryption (backend holds private key) | L4 — don't terminate TLS at LB |
| Clients behind shared NAT | Avoid IP Hash |

---

## Connection to Prior Sessions

- **D07 (Nginx reverse proxy):** Nginx is already an L7 LB. Adding `upstream {}` with multiple servers is just scaling the 1-to-1 reverse proxy to 1-to-many. Same mechanism, more backends.
- **D04 (TLS):** L7 LB terminates TLS — it does the full handshake with the client using its own certificate. It then starts a new TLS connection to the backend. Two separate handshakes.
- **D01 (TCP):** L4 operates at the TCP layer — it's routing TCP connections, not HTTP requests.

---

## Quiz Q&A

**Q1.** You have 3 backends. Server 1 is handling a 30-second video upload. A new request arrives. Which algorithm avoids Server 1, and which might hit it?

**A1.** Least Connections avoids it (Server 1 has the most active connections). Round Robin might send to it (it just cycles, ignores load).

**Q2.** User logs in on Server 2 (session in memory). They refresh and hit Server 3. What happens, and what are the two solutions?

**A2.** Session doesn't exist on Server 3 — user appears logged out. Fix 1: sticky sessions (always route to Server 2). Fix 2: store sessions in Redis (any server can read it). Option 2 is the right long-term fix.

**Q3.** What does an L4 vs L7 LB see for `GET /api/users`?

**A3.** L4: source IP, port, destination IP, port — raw TCP. Cannot see the path. L7: full HTTP request — method, path, headers, body. Must terminate TLS first.
