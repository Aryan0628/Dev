# D03 — HTTP/1.1 vs HTTP/2 vs HTTP/3

**Date completed:** 2026-05-21

---

## The Mental Model

HTTP is a 3-layer stack. The top layer — GET, POST, 200, headers as concepts — never changed across versions. Only the wire format and the transport changed.

```
┌───────────────────────────────────┐
│  HTTP semantics (GET, POST, 200)  │  ← same in all 3 versions
├───────────────────────────────────┤
│  Wire format (text vs binary)     │  ← changes
├───────────────────────────────────┤
│  Transport (TCP or QUIC/UDP)      │  ← changes
└───────────────────────────────────┘
```

---

## The Big Picture

| Feature | HTTP/1.1 (1997) | HTTP/2 (2015) | HTTP/3 (2022) |
|---|---|---|---|
| Transport | TCP | TCP | QUIC (UDP) |
| Format | Text | Binary frames | Binary frames |
| Parallelism | ~6 TCP conns per origin | Multiplexed streams | Multiplexed streams |
| HOL blocking (HTTP layer) | Yes | No | No |
| HOL blocking (transport layer) | Yes | Yes (TCP) | No (QUIC) |
| Header compression | None | HPACK | QPACK |
| Encryption | Optional | De facto required | Mandatory, built-in |
| Connection setup | 2–3 RTT | 2–3 RTT | 1 RTT, often 0 |
| Network change (Wi-Fi → 4G) | Connection drops | Connection drops | Survives |

**RTT** = Round Trip Time = time for a packet to travel to the server and come back.
**HOL** = Head-of-Line blocking = a slow item at the front of a queue blocks everything behind it.

---

## HTTP/1.1 — The Text Protocol

One TCP connection = one conversation at a time. Everything is plain text.

```
Browser ──── TCP conn 1 ────▶ server
              GET /a.jpg  ──▶
              ◀──  response A (500ms)
              GET /b.jpg  ──▶   ← had to WAIT for A to finish
              ◀──  response B
```

Workaround: browsers open ~6 parallel TCP connections per origin to fake parallelism.

**Pain points:**
1. **HOL blocking** — slow request blocks all others on the same connection
2. **Header bloat** — every request resends `Cookie`, `User-Agent`, `Authorization` in full plain text (500–1000 bytes each time)
3. **Connection waste** — 6 × (TCP + TLS handshakes) per page load
4. **No server push** — server cannot preemptively send assets the client will need

Analogy: every employee has 6 personal phone lines to each vendor. Each line needs 30 seconds of dial tone to set up before you can talk.

---

## HTTP/2 — Binary Multiplexing on One TCP Connection

One TCP connection. Many logical streams. Everything is binary frames.

```
Browser ────── ONE TCP connection ──────▶ server
   ↳ Stream 1: GET /a.jpg (slow)
   ↳ Stream 3: GET /b.jpg (fast)
   ↳ Stream 5: GET /c.css

Wire bytes (interleaved):
  [S3 HEADERS] [S1 HEADERS] [S5 HEADERS]
  [S3 DATA "fast resp"]       ← stream 3 comes back first, no waiting
  [S1 DATA ...] [S5 DATA ...]
```

Each frame carries a **stream ID**. Frames interleave on the one TCP connection.

### CRITICAL: "Streams" in HTTP/2 are NOT real transport streams

This is the most important thing to understand. The word "stream" is used at two different layers and means different things:

```
TCP (transport layer):    ONE ordered byte sequence — one pipe
                          ████████████████████████████████

HTTP/2 (app layer):       LOGICAL stream IDs — labels on frames inside that one pipe
                          [S1 frame][S3 frame][S5 frame][S1 frame]...
                               ↑
                          Stream IDs are just numbers in the frame header.
                          TCP has no idea they exist. It just sees bytes.
```

So when a TCP packet drops:

```
TCP pipe:  [S1 frame] [S3 frame] [LOST PACKET] [S5 frame] [S1 frame] ...

TCP rule:  "I cannot give the app ANYTHING past the gap until the lost packet arrives."

Result:    App gets [S1 frame] [S3 frame] → then FROZEN
           S5 frame is sitting in the kernel buffer, ready, but BLOCKED.
```

S5 is physically there, but the kernel won't hand it to the app because TCP requires in-order delivery. The HTTP/2 layer (which is in your app) has zero power to fix this — TCP is in the OS kernel. This is TCP HOL blocking.

---

## HTTP/2 Frame Anatomy

Every byte on an HTTP/2 connection belongs to a frame. 9-byte fixed header:

```
+-------------------------------------------+
| Length (24 bits) | Type (8) | Flags (8) |
+-------------------------------------------+
| R | Stream ID (31 bits)                   |
+-------------------------------------------+
| Frame Payload                             |
+-------------------------------------------+
```

Key frame types:

| Type | Purpose |
|---|---|
| HEADERS | Request/response headers (HPACK compressed) |
| DATA | Body bytes |
| SETTINGS | Connection parameters (max streams, window size) |
| WINDOW_UPDATE | Flow control credit |
| RST_STREAM | Cancel one stream without closing the connection |
| GOAWAY | Graceful shutdown of whole connection |
| PING | Heartbeat / measure RTT |

### Stream IDs

- Client-initiated streams: **odd** numbers (1, 3, 5, …)
- Server-initiated push: **even** numbers (2, 4, …)
- Stream 0: reserved for connection-level control frames
- IDs only go up — never reused on the same connection

---

## HPACK — Header Compression for HTTP/2

Both sides maintain TWO tables:

```
Static table (61 hard-coded entries, same for everyone):
  Index 2  → :method GET
  Index 3  → :method POST
  Index 8  → :status 200
  Index 31 → content-type
  ...

Dynamic table (built up during this connection):
  Index 62 → authorization: Bearer abc123
  Index 63 → content-type: application/json
  Index 64 → user-agent: Mozilla/5.0 ...
```

First request sends the full header string → both sides add it to the dynamic table → every later request just sends the index number (1–3 bytes instead of 30–600 bytes). Typical savings: **85–95% reduction** in header size.

### Why HPACK has a problem

The dynamic table is updated in the order frames arrive on the TCP pipe. This creates a dependency:

```
Stream 5's HEADERS frame says "use index 62 for authorization"
But index 62 was supposed to be added when stream 3's HEADERS arrived.
If stream 3's packet was lost, TCP freezes everything including stream 5.
Stream 5 can't even decode its own headers without waiting for stream 3.
```

This is not just TCP HOL blocking — it's HOL blocking at the header decompression level too, caused by the shared ordered table.

---

## HTTP/3 — Drop TCP Entirely

HTTP/2 killed HOL blocking at the HTTP application layer. But TCP HOL blocking remained. The fix is brutal: stop using TCP.

HTTP/3 runs on **QUIC**, which runs on **UDP (User Datagram Protocol)**.

```
HTTP/1.1, /2:   HTTP → TLS → TCP → IP
HTTP/3:         HTTP → QUIC (TLS built in) → UDP → IP
```

**UDP** = dumb transport. Fires packets with no guarantee of delivery or order. Middleboxes (routers, NATs, firewalls) don't try to inspect or modify UDP the same way they do TCP. This is the key — UDP is neutral ground.

**QUIC** builds everything TCP had — reliability, ordering, congestion control — plus new things, all in userspace (inside the app), not the OS kernel.

### CRITICAL: "Streams" in HTTP/3 ARE real transport streams

Unlike HTTP/2's logical labels on one TCP pipe, QUIC streams are genuinely independent at the transport layer:

```
QUIC connection:
  Stream 1 ─────────────────────────────────────
  Stream 3 ─────────────────────────────────────
  Stream 5 ─────────────────────────────────────
       ↑
  Each stream has its own packet loss recovery.
  A lost packet on stream 3 only stalls stream 3.
```

```
HTTP/2 "streams":  logical labels on ONE TCP pipe → packet loss stalls ALL
HTTP/3 streams:    real independent QUIC streams  → packet loss stalls ONLY its own stream
```

### What QUIC gives you

1. **Independent streams at transport layer** — packet loss only blocks the affected stream
2. **1 RTT connection setup** — merges transport + TLS handshake into one round trip
3. **0-RTT reconnect** — if you visited before, send real data on packet 1 (uses cached crypto keys)
4. **Connection migration** — switch Wi-Fi ↔ 4G mid-request, connection survives
5. **Mandatory built-in encryption** — TLS 1.3 is baked into QUIC, not a separate layer

### Why TCP could not be fixed

Decades of routers, firewalls, and NATs (Network Address Translation boxes) on the internet have TCP assumptions baked in. You can't add new features to TCP without breaking all that middleware. UDP is neutral — middleboxes don't second-guess it. So QUIC implements everything in userspace on top of UDP, where it can evolve freely.

---

## QUIC Handshake

```
Cold connection (never visited):
  Client ── CRYPTO + 0-RTT data ──▶ Server
  Server ── CRYPTO + response ────▶ Client
  Total: 1 RTT

Warm reconnect (visited before, keys cached):
  Client ── 0-RTT data ───────────▶ Server   ← data on first packet
  Total: 0 RTT before real data is sent

Compare:
  TCP + TLS 1.2: 3 RTT before data
  TCP + TLS 1.3: 2 RTT before data
  QUIC cold:     1 RTT before data
  QUIC warm:     0 RTT before data
```

**TLS** = Transport Layer Security — the encryption protocol. **TLS 1.3** is the current version.

---

## Connection Migration — The Mobile Magic

TCP identifies a connection by its **4-tuple**:

```
(source IP, source port, destination IP, destination port)
```

Phone on Wi-Fi: `(192.168.1.5, 51234, 104.21.3.7, 443)`. Walk outside, switch to 4G, IP changes to `10.0.0.2`. 4-tuple changed → TCP says "new connection" → all in-flight requests die.

QUIC puts a **Connection ID (CID)** in every packet:

```
QUIC packet:  [CID: 0xAB12CD34] [stream data...]

On Wi-Fi:     src_IP = 192.168.1.5, CID = 0xAB12CD34
Switch to 4G: src_IP = 10.0.0.2,    CID = 0xAB12CD34  ← same CID

Server: "I know CID 0xAB12CD34 — same connection, IP changed, that's fine."
→ connection survives, requests keep going
```

---

## QPACK — Header Compression for HTTP/3

HPACK has a problem: its shared table is order-dependent (see above). Even in HTTP/3 where QUIC has real independent streams, you could still have a scenario where stream 5's headers reference a table entry that stream 3 was supposed to add — and stream 3's packet is still in flight.

QPACK's fix: separate the table update mechanism from the request/response streams entirely.

```
Two dedicated unidirectional control streams:
  Encoder stream: carries table updates ("add entry: authorization: Bearer abc → index 62")
  Decoder stream: carries acknowledgments ("stream 3 confirmed it received index 62")

Request/response streams (S1, S3, S5):
  Each decodes independently.
  If stream 5's headers don't reference the dynamic table, it decodes immediately.
  If they do, it waits only for that specific entry on the encoder stream — not for stream 3.
```

```
HPACK:  shared table + order-dependent → HOL block at header decompression
QPACK:  shared table + independent decoding via control streams → no HOL block
```

---

## Sockets in HTTP/2 vs HTTP/3

A socket is the OS's API for network communication. It is **not** TCP-specific. There are different types:

```
TCP socket:   SOCK_STREAM   → ordered, reliable byte stream (kernel does all the work)
UDP socket:   SOCK_DGRAM    → fire individual packets, no guarantees (app does the work)
```

HTTP/2 uses a TCP socket. HTTP/3 uses a **UDP socket**. Both use sockets. The difference is what runs on top.

---

## Userspace vs Kernel Space

Every OS divides memory into two zones:

```
┌─────────────────────────────────────────┐
│              KERNEL SPACE               │
│                                         │
│  - Core OS code                         │
│  - TCP/IP stack (tcp.c in Linux kernel) │
│  - Device drivers (network card, disk)  │
│  - Process scheduler                    │
│                                         │
│  Can touch hardware directly.           │
│  Full privileges.                       │
└─────────────────────────────────────────┘
         ↑ protected boundary ↑
┌─────────────────────────────────────────┐
│              USER SPACE                 │
│                                         │
│  - Your app (Python, Node, Go, etc.)    │
│  - nginx, postgres, redis               │
│  - QUIC library (quiche, msquic)        │
│                                         │
│  Cannot touch hardware directly.        │
│  Must ask kernel via syscalls.          │
└─────────────────────────────────────────┘
```

**Syscall** = the only door between userspace and kernel. Example: `send()` = "hey kernel, put these bytes on this socket."

### What OS does vs what app does — TCP vs UDP

```
                TCP                         UDP
                ───                         ───

OS does:  - 3-way handshake             - sends/receives raw packets
          - break data into packets     - that's it
          - number each packet
          - track which arrived
          - retransmit lost packets
          - deliver in order
          - congestion control
          - flow control

App does: - hand bytes to socket        - QUIC: open/close connection
          - read bytes from socket      - QUIC: number packets
          - that's it                   - QUIC: retransmit lost packets
                                        - QUIC: order per stream
                                        - QUIC: congestion control
                                        - QUIC: encryption (TLS built in)
                                        - hand raw UDP packets to socket
```

**TCP** → OS does everything hard. App just sends/receives bytes.
**UDP** → OS does nothing hard. QUIC library in your app does everything TCP's kernel used to do, plus more.

### Why userspace matters for QUIC

```
TCP change → patch submitted to Linux kernel
           → kernel version released
           → server admins update OS (months/years)
           → everyone on old kernels still uses old TCP

QUIC change → update the QUIC library in your app
            → ship new version of nginx/your app
            → done. All users get new behavior immediately.
```

Google shipped QUIC improvements hundreds of times while TCP would have gone through maybe one kernel update cycle. That's the compounding advantage.

---

## Full HTTP/2 Request Flow (Both Sides)

### Step 1: TCP Handshake — 1 RTT

```
CLIENT                              SERVER
  │                                   │
  │──── SYN ────────────────────────▶│   "I want to connect"
  │◀─── SYN-ACK ────────────────────  │   "OK, I'm ready"
  │──── ACK ────────────────────────▶│   "Got it"
  │                                   │
  │  TCP connection open on both sides│
```

**SYN** = Synchronize. **ACK** = Acknowledge. TCP control flags in the packet header. The 3-way handshake takes 1 RTT.

### Step 2: TLS Handshake — 1 more RTT (TLS 1.3)

```
CLIENT                              SERVER
  │                                   │
  │── ClientHello ──────────────────▶│
  │   TLS 1.3, cipher suites,        │
  │   key share,                     │
  │   ALPN: "h2, http/1.1"           │  ← ALPN = Application-Layer Protocol Negotiation
  │                                   │
  │◀── ServerHello + Certificate ─── │
  │    TLS 1.3, cipher chosen,        │
  │    key share, cert,               │
  │    ALPN: "h2"                    │  ← server picked HTTP/2
  │                                   │
  │── Finished ────────────────────▶ │
  │   Both sides have shared keys    │
```

**ALPN** = Application-Layer Protocol Negotiation. A TLS extension where the client lists protocols it supports (h2, http/1.1) and the server picks one. This is how HTTP/2 is agreed on — during TLS, not after.

Total so far: **2 RTT** before any HTTP byte is sent.

### Step 3: HTTP/2 Connection Preface

Both sides must send this immediately after TLS:

```
CLIENT sends:
  "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"   ← magic string hardcoded in spec
  + SETTINGS frame: max streams, window size, max frame size

SERVER sends simultaneously:
  + SETTINGS frame: its own limits
  + SETTINGS ACK: acknowledging client's settings
```

Now real requests can flow.

### Step 4: Actual Requests (multiplexed)

```
CLIENT                              SERVER
  │── HEADERS frame, stream 1 ─────▶│
  │   :method GET, :path /users      │
  │── HEADERS frame, stream 3 ─────▶│   ← same connection, same time
  │   :method POST, :path /orders    │
  │── DATA frame, stream 3 ─────────▶│
  │   {"item": 42}                   │
  │                                   │
  │◀── HEADERS frame, stream 3 ───── │   ← server responded to stream 3 first
  │◀── DATA frame, stream 3 ──────── │       (it finished faster)
  │    {"order_id": 8801}             │
  │◀── HEADERS frame, stream 1 ───── │
  │◀── DATA frame, stream 1 ──────── │
  │    [{"id":1},{"id":2}]           │
```

All of this is on the same TCP connection, same port 443. The OS sees one socket. Stream IDs are invisible to the kernel — the HTTP/2 layer in the app reads them and routes to the right handler.

---

## Full HTTP/3 Request Flow (Both Sides)

### Step 1: QUIC Handshake — 1 RTT (cold start)

QUIC merges transport handshake + TLS 1.3 into one round trip.

```
CLIENT                              SERVER
  │── UDP packet: Initial ─────────▶│
  │   Connection ID: 0xAB12CD34     │   ← random ID client picks
  │   CRYPTO frame: ClientHello     │
  │   key share, ALPN: h3           │
  │                                   │
  │◀── UDP packet: Initial+Handshake │
  │    Connection ID: 0xAB12CD34    │   ← same CID
  │    CRYPTO: ServerHello + Cert   │
  │    key share, Finished          │
  │                                   │
  │── Handshake Finished ───────────▶│
  │                                   │
  │  1 RTT total. Keys derived.      │
  │  No TCP socket anywhere.         │
  │  Just UDP packets back and forth │
```

### Step 2: Control Streams Open

QUIC opens dedicated unidirectional streams for control:

```
CLIENT opens:
  Stream 0: QPACK encoder stream   ← table updates going to server
  Stream 2: QPACK decoder stream   ← acknowledgments going to server
  Stream 4: HTTP/3 SETTINGS

SERVER opens:
  Stream 1: QPACK encoder stream
  Stream 3: QPACK decoder stream
  Stream 5: HTTP/3 SETTINGS
```

**Unidirectional** = flows only one way. These carry control data, not request/response.

### Step 3: Actual Requests

```
CLIENT                              SERVER
  │── QUIC stream 0 (request) ─────▶│
  │   HEADERS: GET /users            │
  │── QUIC stream 4 (request) ─────▶│
  │   HEADERS: POST /orders          │
  │   DATA: {"item": 42}             │
  │                                   │
  │        [packet loss on stream 0] │
  │                                   │
  │◀── stream 4 response ─────────── │   ← unaffected, delivers immediately
  │    :status 201, {"order_id":8801}│
  │                                   │
  │        [stream 0 retransmits]    │
  │◀── stream 0 response ─────────── │
  │    :status 200, [{"id":1}...]    │
```

Stream 4 was never blocked by stream 0's packet loss. Real independent transport streams.

### What the OS sees in HTTP/3

```
Server OS kernel:
  UDP socket on port 443
  Receives raw UDP packets → hands to app
  Done. It does nothing else.

Server app (nginx with QUIC library):
  QUIC layer: reads Connection ID → finds the matching connection object
              reads stream ID → reassembles stream, handles retransmits
  HTTP/3 layer: reads HEADERS frames → routes to handler
  Handler runs → response goes back through QUIC → UDP socket
```

---

## How the Server's Listening Socket Works

The server does **not** open a socket when a client sends a SYN. It opens a listening socket at **startup**, before any client connects, and waits.

```
SERVER STARTUP:
  s = socket(SOCK_STREAM)
  s.bind(("0.0.0.0", 443))    ← claim port 443
  s.listen(128)                ← queue up to 128 pending connections
  while True:
      conn = s.accept()        ← BLOCKS here, sleeping, doing nothing
```

When a SYN arrives:

```
CLIENT                   OS KERNEL (server side)          SERVER APP
  │                              │                              │
  │── SYN ─────────────────────▶│                              │
  │                              │  kernel handles handshake    │
  │◀── SYN-ACK ─────────────────│  automatically               │
  │── ACK ─────────────────────▶│                              │
  │                              │  handshake done              │
  │                              │  kernel creates NEW socket   │
  │                              │  puts it in accept queue     │
  │                              │                        accept() unblocks
  │                              │                        returns new socket
  │                              │                        handler runs
```

The kernel does the entire TCP 3-way handshake. The server app is still sleeping in `accept()` during all of this. By the time `accept()` returns, the connection is fully established.

### Two sockets on the server

```
Listening socket:    0.0.0.0:443        ← ONE, open forever
                                           only job: birth new connection sockets

Connection socket:   server:443 ↔ client_IP:51234   ← NEW one per client
                                           this carries actual request/response data
```

1000 clients connected → server has **1001 sockets** open. 1 listening, 1000 connection sockets.

---

## Handler Routing

Routing is a dictionary lookup. From D02 — still true regardless of HTTP version.

```python
routes = {
    ("GET",  "/users"):   get_users_handler,
    ("POST", "/orders"):  create_order_handler,
}

# Request arrives: GET /users
handler = routes[("GET", "/users")]
result  = handler(request)
```

`@app.get("/users")` in FastAPI just does: `routes[("GET", "/users")] = get_users`. That's it.

In HTTP/2 and HTTP/3, routing happens after the stream is reassembled:

```
HTTP/2:
  TCP bytes → kernel → app → HTTP/2 layer reassembles stream by stream ID
  → router does (method, path) lookup → handler runs

HTTP/3:
  UDP packets → kernel → app → QUIC layer reassembles stream
  → HTTP/3 layer reads headers → router does (method, path) lookup → handler runs
```

The router is identical in all versions. The transport version only affects how bytes reach the router.

---

## How a Browser Discovers HTTP/3

```
Step 1: First visit — browser doesn't know server supports HTTP/3.
        Connects over TCP, HTTP/2.

Step 2: Server responds with header:
        Alt-Svc: h3=":443"; ma=86400
        (Alt-Svc = Alternative Services)
        "I also speak h3 on port 443. Remember this for 86400 seconds."

Step 3: Browser caches this. Next visit, tries UDP port 443 directly.
        Works → HTTP/3. UDP blocked (common in corp networks) → falls back to HTTP/2.
```

---

## Config — Enabling HTTP/2 and HTTP/3 in nginx

### HTTP/2 (one line change)

```nginx
server {
    listen 443 ssl http2;        # ← just add "http2"
    server_name api.example.com;
    ssl_certificate     /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;
}
```

ALPN handles negotiation automatically. Browsers that don't support HTTP/2 fall back to /1.1.

### HTTP/3

```nginx
server {
    listen 443 ssl;
    listen 443 quic reuseport;       # ← UDP 443 for QUIC
    http2 on;
    http3 on;
    add_header Alt-Svc 'h3=":443"; ma=86400';   # ← tells browsers HTTP/3 is available
}
```

### Inspect from terminal

```bash
# Check which HTTP version is negotiated
curl --http1.1 -v https://example.com
curl --http2   -v https://example.com   # look for "Using HTTP2, server supports multiplexing"
curl --http3   -v https://example.com

# See ALPN negotiate the protocol during TLS
openssl s_client -connect example.com:443 -alpn h2,http/1.1 | grep ALPN
# → ALPN protocol: h2
```

---

## When NOT To Use Each

### HTTP/2

| Use it when | Stick with /1.1 when |
|---|---|
| Page loads many small assets | One huge file download (no parallelism benefit) |
| Mobile clients | Behind ancient middleware that doesn't handle /2 |
| You control the load balancer | Debugging — text protocol is easier to eyeball |

Hidden gotcha: On a very lossy network, HTTP/2's single TCP connection can be **slower** than HTTP/1.1's six connections, because TCP retransmissions block all streams. (This is exactly what HTTP/3 fixes.)

### HTTP/3

| Use it when | Skip it when |
|---|---|
| Mobile / lossy network users | Corporate clients (UDP/443 often blocked at firewall) |
| Latency-sensitive (gaming, real-time, live video) | Internal datacenter traffic (reliable network, TCP fine) |
| Already on a CDN that supports it (Cloudflare, Fastly) | You'd have to roll your own QUIC stack |

CPU cost: QUIC does crypto + reliability in userspace — about 2× more CPU per byte than TCP+TLS. Large-scale servers notice this.

### Rule of thumb

- Public-facing edge (browser → CDN, browser → your server) → biggest wins from /2 and /3
- Internal east-west microservices → HTTP/1.1 keep-alive is usually fine
- gRPC (microservice framework) requires HTTP/2 — that's one concrete reason to upgrade internal traffic

---

## Production Gotchas

| Situation | What goes wrong |
|---|---|
| HTTP/2 on lossy network | One TCP loss stalls all streams — can be slower than /1.1 |
| HTTP/3 in corporate network | UDP 443 blocked → silent fallback to /2, first request slow |
| Server push (HTTP/2) | Usually hurts more than helps; Chrome removed support |
| Forgetting `Alt-Svc` header | Browsers never discover your HTTP/3 endpoint |
| Listening on 127.0.0.1 only | Not reachable from outside the machine |
| Backlog too small + slow accept() | New connections refused before your code even runs |

---

## Key Terms Glossary

| Term | Full form / meaning |
|---|---|
| RTT | Round Trip Time — time for packet to go to server and come back |
| HOL | Head-of-Line blocking — slow item at front of queue blocks everything behind it |
| TCP | Transmission Control Protocol — reliable, ordered byte stream, kernel-implemented |
| UDP | User Datagram Protocol — fire-and-forget packets, no ordering/reliability |
| QUIC | Not an acronym — Google's protocol; transport for HTTP/3, runs on UDP |
| TLS | Transport Layer Security — encryption protocol (TLS 1.3 is current) |
| ALPN | Application-Layer Protocol Negotiation — TLS extension to agree on HTTP version |
| HPACK | Header compression for HTTP/2 — shared ordered table |
| QPACK | Header compression for HTTP/3 — shared table, independently decodable streams |
| Alt-Svc | Alternative Services — HTTP header advertising HTTP/3 availability |
| CID | Connection ID — QUIC's connection identifier; survives IP changes |
| 0-RTT | Zero Round Trip Time — send data on first packet of a reconnect |
| SYN | Synchronize — TCP handshake flag: "I want to connect" |
| ACK | Acknowledge — TCP flag: "I received that" |
| SOCK_STREAM | Socket type for TCP (ordered stream) |
| SOCK_DGRAM | Socket type for UDP (individual datagrams/packets) |
| Syscall | System Call — the only door between userspace and kernel |
| Listening socket | Server socket open at startup; births new sockets per client |
| Connection socket | The actual TCP/QUIC connection to one specific client |
| NAT | Network Address Translation — device that maps private IPs to one public IP |
| Middlebox | Any network device between client and server (router, firewall, NAT) |
| Unidirectional stream | QUIC stream that flows only one way — used for QPACK control |

---

## Quiz

1. **You're loading a page with 50 small images from the same origin. The network has 5% packet loss. Which is likely fastest: HTTP/1.1 with 6 connections, HTTP/2 on one connection, or HTTP/3? Why?**

2. **Your microservices talk to each other inside a datacenter over a private VPC (very low latency, zero packet loss). Is there a strong reason to upgrade them from HTTP/1.1 keep-alive to HTTP/2 or HTTP/3? What if they used gRPC?**

3. **A user opens your site on Wi-Fi, the page starts loading, then they walk outside and their phone switches to 4G mid-load. Describe what happens to the in-flight requests on HTTP/2 vs HTTP/3.**
