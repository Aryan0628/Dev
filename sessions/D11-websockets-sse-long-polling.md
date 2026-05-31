# D11 — WebSockets vs Long Polling vs SSE: When to Use Which

**Date:** 2026-05-30
**Phase:** 1 — Foundations

---

## Why It Matters

HTTP is request-response — the client asks, the server answers. That model breaks the moment the **server needs to push data to the client without the client asking first**: chat messages arriving, live prices updating, collaborative edits from another user, real-time game state.

The naive fix is short polling — the client asks every second, "anything new?" Three better solutions exist, each solving the problem at a different point in the stack.

---

> **Real case: Slack — 2015–2016 (WebSocket thundering herd)**
>
> **What happened:** When iOS/Android users switched networks (WiFi → cellular) or unlocked their phones after a meeting, all disconnected clients tried to reconnect their WebSockets simultaneously.
>
> **Root cause:** No reconnection jitter. All clients used the same fixed retry delay — a network event triggered a simultaneous mass reconnect spike → server overloaded → connections failed → clients retried → loop.
>
> **Cost / impact:** Degraded service during high-reconnect windows. Documented internally as "thundering herd."
>
> **What it would have taken to prevent it:** Exponential backoff + jitter (random delay added to retry) — standard practice now but commonly skipped.

---

## The Analogy

| Technique | Analogy |
|---|---|
| Short polling | You walk to the front door every 30 seconds to check for a package. |
| Long polling | You go to the door and wait there. Delivery person brings the package to you, then you go back inside and immediately return to wait again. |
| SSE | You open the mail slot. Letters come through whenever the sender has something. You never close the slot. |
| WebSocket | You and the delivery person each have a walkie-talkie. Either of you can talk at any time. |

---

## How Each One Works Internally

### Long Polling **[IMPORTANT]**

Standard HTTP, but the server **holds the response open** until it has something to send (or a timeout fires).

```
Client                              Server
  |                                   |
  |  GET /messages?since=t123 ------> |
  |                                   | (parks the request, waiting for an event)
  |                                   | (new message arrives at t=8s)
  | <----- 200 OK {message: ...} ---- |
  |                                   |
  |  GET /messages?since=t456 ------> |  ← client immediately makes a new request
```

**Under the hood:**
- The TCP connection stays open. The HTTP request is pending.
- The server parks the request in an event loop / queue, waiting for a database notification, pub/sub event, or timeout.
- When the event fires, the server writes the HTTP response and the connection cycle ends.
- The client immediately opens a new long-poll request — so there is always exactly one open connection.

**What breaks:**
- **Proxy timeouts** — Nginx (D07) default idle timeout is 60–75s. If no message arrives before that, the proxy kills the connection silently. The server must send a "nothing yet, reconnect" response before the timeout fires.
- **One pending request per client** — in HTTP/1.1, you can't send other requests on the same connection while a long-poll is pending.
- **Header overhead** — every cycle sends full HTTP request + response headers. At thousands of clients, this adds up.

---

### SSE — Server-Sent Events **[IMPORTANT]**

A **single, long-lived HTTP response** where the server keeps writing data without ever closing the connection.

**The trick:** HTTP/1.1 chunked transfer encoding allows the server to send a response body in pieces over time. SSE exploits this.

**What the response looks like:**

```
HTTP/1.1 200 OK
Content-Type: text/event-stream     ← this MIME type tells the browser: expect a stream
Cache-Control: no-cache
Connection: keep-alive

data: {"price": 142.30}

data: {"price": 142.55}

data: {"price": 141.90}
```

Events are separated by blank lines (`\n\n`). The browser's built-in `EventSource` API parses this automatically.

```
Client                              Server
  |                                   |
  |  GET /prices/AAPL (SSE) -------> |
  | <---- 200 OK (headers only) ----- |
  | <---- data: {price:142.3}\n\n --- |  ← pushed whenever price changes
  | <---- data: {price:142.5}\n\n --- |
  | <---- data: {price:141.9}\n\n --- |  (connection stays open indefinitely)
```

**Under the hood:**
- Same TCP connection, same HTTP response — the response body is never finished (no Content-Length).
- The kernel's TCP stack (D06) flushes each `data:` chunk to the client as the server writes it.
- `EventSource` fires a JavaScript `message` event for each chunk — no parsing code needed.

**Built-in reconnect:** If the connection drops, `EventSource` **automatically reconnects** and sends a `Last-Event-ID` header so the server can resume where it left off.

**SSE is one-directional: Server → Client only.** For client → server, the client makes normal HTTP requests.

**SSE + HTTP/2 (D03):** In HTTP/1.1, each SSE stream needs its own TCP connection (browsers cap at ~6 per domain). In HTTP/2, multiple SSE subscriptions multiplex over a single TCP connection as separate HTTP/2 streams — major win.

---

### WebSocket **[CRITICAL]**

A **separate protocol** (`ws://` or `wss://` for encrypted). It starts as HTTP and upgrades mid-connection.

**The upgrade handshake:**

```
Client                              Server
  |                                   |
  |  GET /chat HTTP/1.1 -----------> |
  |  Upgrade: websocket              |  ← client requests upgrade
  |  Connection: Upgrade             |
  |  Sec-WebSocket-Key: dGhlIHNhb... |  ← random 16-byte key, base64 encoded
  |  Sec-WebSocket-Version: 13       |
  |                                   |
  | <---- HTTP/1.1 101 Switching Protocols ----
  |  Upgrade: websocket              |  ← server accepts
  |  Sec-WebSocket-Accept: s3pPLMB... |  ← server hashes key + magic string as proof
  |                                   |
  | [from here: WebSocket frames, not HTTP]
  | <======= ws frames ============> |  ← fully bidirectional
```

**Status code 101** = "I'm done with HTTP now; the TCP connection below is yours to use as WebSocket."

**The frame format (under the hood):**

After the handshake, both sides send **WebSocket frames** — not HTTP messages. A frame has:
- 2-byte header: opcode (text, binary, ping, pong, close), FIN bit (last frame of message?), MASK bit
- Optional payload length (7 bits / 16-bit / 64-bit extended)
- Optional 4-byte masking key — **client → server frames MUST be masked** (spec rule to prevent cache poisoning on shared proxies); server → client frames are NOT masked
- Payload

A WebSocket "message" can be split across multiple frames (fragmentation) and reassembled on the other end.

**Ping/Pong keepalive:** WebSocket has built-in ping/pong frames. Either side sends Ping, the other must reply Pong. Critical: many proxies/load balancers have idle timeouts (Nginx default: 65s). Without ping/pong, the proxy silently kills the TCP connection and both sides have a **zombie connection** — each thinks the other is alive.

**Full duplex:** Either side can send a frame at any time without waiting. This is the key difference from SSE — the client can also push messages over the same connection.

**Load balancing (D08):** WebSocket is stateful. Once a client connects to server instance A, all messages must go to A. If the load balancer routes a later frame to server B, B has no context. Two fixes:
1. **Sticky sessions** (session affinity) at the load balancer — all requests from a given client go to the same server instance.
2. **Shared pub/sub bus** (e.g. Redis pub/sub) — any server instance can handle any client; messages are broadcast via Redis so all instances are in sync.

---

## Side-by-Side Comparison

| | Long Polling | SSE | WebSocket |
|---|---|---|---|
| Direction | Server → Client | Server → Client | **Bidirectional** |
| Protocol | HTTP | HTTP | **ws:// (its own protocol)** |
| Connection | New request each cycle | One long-lived HTTP response | Persistent connection after upgrade |
| Latency | Medium (one RTT per cycle) | Low | Lowest |
| Client sends data | New HTTP request | New HTTP request | **Same connection** |
| Auto-reconnect | Manual (app code) | **Built-in** (EventSource) | Manual (must implement) |
| HTTP/2 benefit | Slight | **Major** (multiplexing) | Not applicable |
| Load balancing | Stateless, easy | Stateless, easy | **Needs sticky sessions or shared bus** |
| Proxy-friendly | Yes | Yes | Needs explicit proxy config |
| Browser support | Universal | Universal (no IE) | Universal |

---

## When to Use Which

**Use Long Polling when:**
- Infrequent updates (every 10–60s) — overhead per cycle is acceptable
- You can't control infrastructure (some enterprise proxies block WebSocket upgrades)
- Simplest possible implementation with no special server setup needed

**Use SSE when:** **[IMPORTANT]**
- Data only flows server → client (live feeds, notifications, progress bars, log tailing)
- You're on HTTP/2 (multiplexing win)
- You want built-in reconnect and event ID resumption for free
- Data is text-based

**Use WebSocket when:** **[CRITICAL]**
- True bidirectional, low-latency communication (chat, collaborative editing, live games, trading)
- Client needs to push data frequently over the same connection
- Sub-100ms round trips matter

---

## Trade-offs & Common Production Mistakes

**Mistake #1 — using WebSockets for one-way feeds** **[IMPORTANT]**
A live price ticker, a notification system, a log stream — these are server → client only. WebSocket adds upgrade handshake, ping/pong management, sticky session requirements, and manual reconnect logic for zero benefit over SSE. SSE handles this with two lines of code.

**Mistake #2 — no reconnect logic for WebSockets** **[CRITICAL]**
WebSocket connections drop. Mobile networks switch. Phones sleep. Every WebSocket client must implement:
1. `onclose` → reconnect with **exponential backoff + jitter** (random delay) — the Slack incident above
2. Server-side ping/pong to detect zombie connections (clients that didn't close cleanly)
3. State reconciliation on reconnect — the client missed N messages while disconnected; how does it catch up?

**Mistake #3 — WebSockets through unconfigured proxies**
Old Nginx configs or corporate proxies strip the `Upgrade` header silently. The WebSocket handshake fails, the client gets a confusing error, and HTTP never upgrades. Fix: explicitly pass through `Upgrade` headers in Nginx config (see below).

**When NOT to use WebSockets:**
Any CRUD-heavy app where data flows are mostly request-response. Adding WebSockets to a blog or to-do app is pure complexity. Stateful connections, sticky sessions, and reconnect handling are not worth it when you don't need push at all.

---

## Code / Config Examples

**SSE — Server (Node.js):**
```javascript
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const send = (data) => res.write(`data: ${JSON.stringify(data)}\n\n`);

  const interval = setInterval(() => send({ price: Math.random() }), 1000);
  req.on('close', () => clearInterval(interval)); // clean up when client disconnects
});
```

**SSE — Client (browser, one line):**
```javascript
const es = new EventSource('/events');
es.onmessage = (e) => console.log(JSON.parse(e.data));
// auto-reconnects on drop — nothing else needed
```

**WebSocket — Server (Node.js, ws library):**
```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  ws.on('message', (msg) => {
    // broadcast to all connected clients
    wss.clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) client.send(msg);
    });
  });
  ws.ping(); // start ping/pong keepalive
});
```

**Nginx config for WebSocket proxying — critical extra lines:**
```nginx
location /ws {
  proxy_pass http://backend;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;      # pass through the Upgrade header
  proxy_set_header Connection "upgrade";       # required for Nginx to allow protocol upgrade
  proxy_read_timeout 3600s;                    # don't kill idle WebSocket connections after 65s
}
```

Without the two `proxy_set_header` lines, Nginx strips the `Upgrade` header and the WebSocket handshake silently fails — the client gets an HTTP 200 or error instead of 101.

---

## Connections to Previous Sessions

- **D01 (TCP)** — All three techniques run over TCP. WebSocket uses TCP directly after the HTTP upgrade is complete.
- **D03 (HTTP/2)** — SSE over HTTP/2 multiplexes many event streams over one TCP connection via separate HTTP/2 streams — the HTTP/1.1 browser 6-connection-per-domain limit no longer applies.
- **D06 (Async I/O)** — SSE server loops and WebSocket servers use epoll (Model 3) to hold thousands of open connections on a small thread pool without blocking.
- **D07 (Nginx)** — The Nginx config above extends what you learned about reverse proxying. Without `Upgrade` header passthrough, WebSocket breaks silently.
- **D08 (Load Balancing)** — WebSocket requires sticky sessions or a shared pub/sub bus. SSE and Long Polling are stateless — any server instance can handle any client.

---

## Quiz

**Q1:** You're building a live dashboard that shows server CPU and memory metrics, updated every second, displayed in a browser. Data flows one way (server → client). Which technique do you choose and why?

**A:** SSE. It's server → client only, so bidirectionality of WebSocket is wasted complexity. SSE has built-in reconnect, works over HTTP (no protocol upgrade needed), and gets HTTP/2 multiplexing for free. Long polling adds one RTT of extra latency per update cycle and re-opens a connection every second — wasteful.

---

**Q2:** A WebSocket connection through Nginx drops exactly at 65 seconds of no messages. What's the cause and how do you fix it?

**A:** Nginx's default `proxy_read_timeout` is 65s. An idle WebSocket connection with no frames in 65s looks like a stalled request to Nginx — it kills it. Two fixes: (1) increase `proxy_read_timeout` in the Nginx location block (e.g., 3600s), and (2) implement server-side WebSocket ping/pong so a frame travels before the timeout fires — keeping the connection alive regardless of application-level silence.

---

**Q3:** Your chat app uses WebSockets on 3 backend instances behind a round-robin load balancer. Users report messages aren't reaching everyone. What's the root cause?

**A:** Sticky sessions are missing. Client A connects to Instance 1. Client B connects to Instance 2. Client A sends a message → Instance 1 only broadcasts to clients connected to it → Client B on Instance 2 never receives it. Fix: either (1) sticky sessions so users sharing a "room" land on the same instance, or (2) a shared Redis pub/sub — when Instance 1 receives a message, it publishes to Redis; all instances subscribe and broadcast to their local clients.

---

## Follow-up Deep Dives

### Long Polling + the Three Concurrency Models — how "parked" actually works

When the server holds a long poll request open waiting for data, what happens depends entirely on which concurrency model (D06) the server uses.

**Model 1 & 2 — Process/Thread per connection**

A thread (or process) is assigned to the request. It calls `queue.get()` or `select()` on an empty socket. The OS moves it to **BLOCKED state** — same as sleeping. Timer does not fire on it. Zero CPU consumed. But it holds memory: ~1–8MB per thread stack.

```
Connection 1 → Thread 1 → BLOCKED (sleeping, waiting for its event)
Connection 2 → Thread 2 → BLOCKED
...
Connection 10,000 → Thread 10,000 → BLOCKED
10K threads × ~8MB = ~80GB RAM — dead
```

**Model 3 — Event loop (Node.js)**

No thread is assigned. The connection is registered with epoll and a callback stored in heap. The single thread stays free to handle other requests. When nothing is ready, the thread calls `epoll_wait()` — ONE thread sleeping for ALL connections simultaneously.

```
Connection 1 → fd in epoll + heap callback  (no thread sleeping for it)
Connection 2 → fd in epoll + heap callback
...
Connection 10,000 → fd in epoll + heap callback

Event loop thread → epoll_wait([all 10,000 fds])  ← 1 thread sleeping for everyone
```

The kernel watches all 10,000 fds at once. The moment any one has data, `epoll_wait()` returns, the event loop runs that connection's callback.

| | Who sleeps | Sleepers for 10K connections |
|---|---|---|
| Thread per connection | Each thread on its own fd | 10,000 |
| Event loop | One thread on epoll_wait (all fds) | 1 |

**BLOCKED = sleeping = CPU timer off = zero CPU consumed.** The cost difference is memory: 10K thread stacks vs 10K small heap objects (~KB each).

---

### Node.js / Async Event Loop — full rewind

**The three pieces:**

**1. Non-blocking I/O** — sockets opened in non-blocking mode. `read()` on an empty socket returns immediately with EAGAIN ("not yet") instead of sleeping. Node never blocks on a single connection.

**2. epoll** — kernel data structure. You register many file descriptors with it. `epoll_wait()` blocks until ANY of them has data, then returns which ones fired. The kernel watches all fds simultaneously in hardware.

**3. The event loop** — a while loop:
```
while (true) {
    1. Run all callbacks that are currently ready
    2. Nothing ready → epoll_wait()  ← thread BLOCKS, timer off
    3. OS: "fd47 has data"
    4. Queue callback for fd47
    5. Back to step 1
}
```

**What `await` actually does:**
```javascript
async function handleRequest() {
    const result = await db.query("SELECT ...");
    // 1. sends SQL bytes to DB socket (non-blocking, instant)
    // 2. registers socket with epoll: "wake me when response arrives"
    // 3. pauses THIS function, hands thread back to event loop
    // 4. event loop runs other callbacks
    // 5. DB responds → epoll fires → this line resumes
    res.send(result);
}
```

`await` pauses the function, not the thread. The function is a **coroutine** — it can pause and resume mid-execution. The thread stays free.

**What kills the event loop:**
```javascript
async function bad() {
    resizeImage(bigFile); // CPU work, never hits await
    // thread stuck here for 2 seconds
    // ALL other connections frozen
}
```
If code never hits `await`, it never yields. Event loop can't run anything else. Fix: offload CPU work to a worker thread/process, then `await` the result.

**Rule:** async wins for I/O-bound work. CPU-bound work must be offloaded.

---

### WebSocket Internals — deep dive

#### The upgrade handshake

`Sec-WebSocket-Key` is a random 16-byte value (base64 encoded). The server:
1. Takes the key
2. Appends the magic string `"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"` (hardcoded in RFC 6455 — a UUID chosen to never appear in any other protocol)
3. SHA-1 hashes the combined string
4. Base64-encodes the result → sends as `Sec-WebSocket-Accept`

**Why:** proves the server genuinely understands WebSocket. Prevents a plain HTTP server or proxy from accidentally accepting the upgrade. Client verifies the hash matches before proceeding.

After the 101 response, both sides drop HTTP completely. The same TCP socket fd is now used for raw WebSocket frames.

#### Frame format — bit layout

```
Byte 0          Byte 1
┌─┬─┬─┬─┬──────┬─┬──────────┐
│F│R│R│R│opcode│M│payload   │ + optional extended length
│I│S│S│S│      │A│length    │ + optional 4-byte masking key
│N│V│V│V│      │S│(7 bits)  │ + payload bytes
└─┴─┴─┴─┴──────┴─┴──────────┘
```

**FIN bit:** 1 = last (or only) frame of message. 0 = more frames follow (fragmentation).

**Opcodes:**
| Code | Meaning |
|---|---|
| 0x0 | Continuation frame |
| 0x1 | Text (UTF-8) |
| 0x2 | Binary |
| 0x8 | Close |
| 0x9 | Ping |
| 0xA | Pong |

**Payload length encoding:**
- 0–125 → that value IS the length
- 126 → next 2 bytes hold the real length (up to 65KB)
- 127 → next 8 bytes hold the real length (up to 18 exabytes)

Small messages stay cheap — a 10-byte message only needs 2 header bytes total.

**Masking — why client→server must be masked, server→client must NOT:**

Masking is XOR, not encryption. The 4-byte masking key is in the frame — anyone can unmask it. The purpose is to prevent **cache poisoning on transparent proxies**.

If a proxy doesn't understand WebSocket and the client could send unmasked bytes that looked like an HTTP request, the proxy might cache a fake response and serve it to other users. By forcing client→server data to be XOR-masked with a fresh random key each frame, the payload is randomized — impossible to craft into a fake HTTP request. Server→client doesn't need masking because the server controls its output.

#### Ping/Pong — zombie connection problem

**How a zombie forms:**
1. Client's phone drops signal abruptly — no FIN packet sent
2. Server's TCP sees no FIN → thinks connection alive
3. Server has open fd in epoll, callback registered, memory allocated
4. No data ever arrives — epoll never fires for this fd
5. Zombie: consuming resources, no live client on the other end

**Fix:** server sends Ping frame every 25–30s. If no Pong within timeout → close fd, free resources.

TCP keepalive exists at OS level (`SO_KEEPALIVE`) but defaults to 2 hours — useless. WebSocket ping/pong gives application-level keepalive with any interval you want.

---

### Two types of cookies — do not confuse these

**Sticky session cookie (load balancing):**
- Value: `SERVERID=instance1`
- Stored: client browser
- Read by: load balancer
- Purpose: route this client to the same server instance
- Redis: NOT involved

**Auth session cookie (D09):**
- Value: `sid=a7f3b2` (a random meaningless key)
- Stored: client browser
- Read by: backend server
- Purpose: identify who the user is
- Redis: holds the actual user data `{ "a7f3b2": { user_id: 42 } }`

Both use a cookie on the client. The sticky session cookie tells the **load balancer where to route**. The auth session cookie tells the **server who you are** — the data lives in Redis.

If you use auth sessions across multiple servers, you need either sticky sessions (fragile — instance dies, user logged out) OR shared Redis that all instances can reach (correct approach — any instance looks up the same Redis).

---

### Redis pub/sub — how cross-instance WebSocket fan-out works

**The problem:** Each server instance holds WebSocket connections in its own memory. Instance 1 has Client A. Instance 2 has Client B. Both are in room 42. Client A sends "hello" — Instance 1 has no way to reach Client B.

**Redis pub/sub analogy:** a radio station. Publisher broadcasts on a frequency. Every subscriber tuned to that frequency hears it simultaneously. Redis channels are the frequencies.

**Two operations:**
- `SUBSCRIBE room:42` — "push me every message published here"
- `PUBLISH room:42 "hello"` — "send this to all subscribers now"

All three instances subscribe to `room:42` at startup. One persistent TCP connection each to Redis.

**Full flow when Client A sends "hello room 42":**
```
Client A → Instance 1 (WebSocket frame)
Instance 1 → Redis: PUBLISH room:42 {msg:"hello", from:"aryan"}
Redis → Instance 1, 2, 3 simultaneously (all subscribed)
Each instance: checks local memory for clients in room:42 → sends WebSocket frame to each
```

Instance 1 doesn't know where Client B is. It doesn't need to. Redis handles the fan-out.

**What each instance stores — only its own connections:**
```
Instance 2 memory:
  rooms: { "room:42": [fd82, fd88] }
  clients: { fd82: { userId: "clientB", ws: <ws object> } }
```

**On reconnect:** Client B drops, reconnects to Instance 3. Instance 3 adds fd105 to its local `rooms["room:42"]`. Done — Redis didn't care, channel still exists.

**Note: two Redis connections needed per instance** — one for subscribing (always listening, can't also publish), one for publishing.

**When this breaks at scale:** Every instance receives every message for every room, even rooms with zero local clients. At 10K rooms × 100 instances, most messages are received and discarded. Fix: Kafka, partitioned by room ID, so each instance only consumes messages for rooms it actually serves. This is why Discord/Slack moved from Redis pub/sub to Kafka at large scale.

---

### Subscribe vs Publish — the two roles, and why two Redis connections per instance

**PUBLISH** = send a message into a channel. Fire and forget. The publisher doesn't know who's listening.

**SUBSCRIBE** = register to receive all future messages on a channel. Redis pushes messages to you whenever anyone publishes.

**The terminology trap — "client" means two different things here:**

| Term | Means |
|---|---|
| **Redis client** | A library/connection your Node.js code uses to talk to Redis (e.g. `ioredis`) |
| **WebSocket client** | A browser user connected to your server |

These are completely unrelated. A Redis client is just a TCP connection from your backend to Redis.

**Why each server instance needs two Redis clients:**

Once you call SUBSCRIBE on a Redis connection, that connection enters a streaming receive mode — locked into listening. The Redis protocol does not allow you to send any other commands (including PUBLISH) on the same connection. Calling PUBLISH on a subscribed connection gets rejected.

```
Instance 2 (Node.js app)
  │
  ├── redisSubscriber (Redis client #1)
  │     → called SUBSCRIBE room:42, room:99...
  │     → Redis pushes messages here whenever anyone publishes to those rooms
  │     → LOCKED — can only receive, cannot send commands
  │
  └── redisPublisher (Redis client #2)
        → called when a WebSocket client sends a message: PUBLISH room:42 "hello"
        → also used for regular Redis commands (GET, SET, etc.)
```

Same instance both listens (via subscriber connection) and sends (via publisher connection). Two TCP connections to Redis, two roles.

---

### Revisit Quiz — Redis pub/sub & cross-instance scaling

**Q1:** You have 5 server instances. All subscribe to `room:42`. Client A on Instance 2 sends a message. How many instances receive the Redis PUBLISH event, and what does each one do with it?

**A:** All 5 instances receive the PUBLISH event. Each one checks its local room map — instances that have clients in room:42 send the WebSocket frame to those clients. Instances with no local clients in room:42 discard it.

---

**Q2:** A junior engineer says "we only need one Redis connection per instance — we can subscribe and publish on the same connection." What's wrong with this and how does it break in production?

**A:** A subscribed connection is locked in listen-only mode by the Redis protocol. Calling PUBLISH on it gets rejected. In production: the first time the app tries to publish, it errors or hangs. You need two separate connections — one dedicated to SUBSCRIBE (always listening), one for PUBLISH and other commands.

---

**Q3:** Your chat app has 50 instances and 100,000 active rooms. Most rooms have clients on only 1–2 instances. You notice Redis CPU spiking and all instances have high "discard" rates on pub/sub. What architectural change do you make and why?

**A:** Switch to Kafka partitioned by room ID. Each partition is assigned to one instance — only that instance receives messages for rooms in its partition. Redis pub/sub sends every message to all 50 instances regardless of whether they have local clients in that room; at this scale, most deliveries are wasted. Kafka eliminates the fan-out waste by routing messages only to the instance that owns that room's partition.
