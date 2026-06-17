# D43 — gRPC vs REST: Protobuf, Streaming, When to Prefer gRPC

---

## Why It Matters

REST + JSON works for browser-to-server. But with 20 microservices (D42) calling each other thousands of times/sec: JSON is text (slow parsing), no schema enforcement (runtime errors), no streaming (request→response only). gRPC solves all three.

> **Real case: Google — Stubby → gRPC**
> Google built Stubby internally for billions of inter-service calls/sec. REST + JSON too slow. Binary encoding, schema contracts, HTTP/2 multiplexing. Open-sourced as gRPC in 2015.

> **Real case: Netflix — REST → gRPC for inter-service**
> 700+ microservices using REST + JSON. JSON parsing consumed measurable CPU across the fleet. Migrated critical paths to gRPC + Protobuf → ~30% reduction in serialization CPU.

---

## What Is gRPC? [CRITICAL]

**gRPC** = gRPC Remote Procedure Call. An **RPC (Remote Procedure Call)** makes calling a remote server feel like a local function call:

```
REST:  response = http.post("http://user-service/users", body=JSON)

gRPC:  user = userService.getUser(userId=123)   ← feels like local function
       (library handles serialization, network, errors)
```

**Still a network call** — same latency and failure risks as HTTP. gRPC hides the plumbing.

---

## Pillar 1: Protocol Buffers (Protobuf) [CRITICAL]

### The .proto File — The Contract

```protobuf
syntax = "proto3";

message User {
  int32 id = 1;          // field NUMBER, not name
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  int32 user_id = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(User) returns (User);
}
```

Auto-generate code from one `.proto` file for any language:
```
user.proto → protoc → Python, Go, Java, C++, JS client/server code
```

Both client and server generate from same `.proto` → guaranteed agreement on data format.

### JSON vs Protobuf on the Wire

```
JSON:     {"id": 12345, "name": "Aryan", "email": "aryan@example.com"}
          58 bytes, text, field names sent every time

Protobuf: 08 B9 60 12 05 41 72 79 61 6E 1A 11 ...
          30 bytes (~50% smaller), binary, only field numbers sent
```

Field names never sent — both sides have the `.proto` file, know field 1 = `id`, field 2 = `name`.

| Dimension | JSON (REST) | Protobuf (gRPC) |
|---|---|---|
| Format | Text | Binary |
| Size | Larger (names repeated) | ~50-80% smaller |
| Parsing | Slow (string parsing) | Fast (direct memory read) |
| Human readable | ✅ | ❌ |
| Schema | Optional | **Required** |
| Type safety | Runtime errors | **Compile-time errors** |

---

## Pillar 2: HTTP/2 Transport [CRITICAL]

gRPC runs on HTTP/2 (from D03). REST typically uses HTTP/1.1.

```
REST (HTTP/1.1):  one request per connection (or need multiple connections)
gRPC (HTTP/2):    many RPCs on ONE connection, multiplexed

  Connection 1: ──→ GetUser(1)    ──→ response  ┐
                ──→ GetOrder(5)   ──→ response  ├─ all interleaved
                ──→ ChargeCard()  ──→ response  ┘
```

Thousands of RPCs/sec on a single TCP connection — reuses same connection from D01.

---

## Pillar 3: Streaming [IMPORTANT]

REST: one request → one response. gRPC: four patterns.

### 1. Unary (like REST) [CRITICAL]
```
Client ── GetUser(123) ──→ Server
Client ←── User{...} ───── Server
```
One request, one response. Majority of gRPC calls.

### 2. Server Streaming [IMPORTANT]
```
Client ── ListOrders(user=123) ──→ Server
Client ←── Order{1} ──────────── Server
Client ←── Order{2} ──────────── Server
Client ←── Order{3} ──────────── Server
Client ←── (stream ends) ────── Server
```
Use case: Large result sets. Stream results as found instead of loading all into memory. Like SSE (D11) but binary over HTTP/2.

### 3. Client Streaming [GOOD TO KNOW]
```
Client ── Metric{cpu=80%} ──→ Server
Client ── Metric{cpu=85%} ──→ Server
Client ── (done) ───────────→ Server
Client ←── Summary{avg=82%} ── Server
```
Use case: IoT telemetry, log upload.

### 4. Bidirectional Streaming [GOOD TO KNOW]
```
Client ── Message{"hi"} ────→ Server
Client ←── Message{"hello"} ── Server
Client ── Message{"bye"} ───→ Server
```
Like WebSockets (D11) but with protobuf + HTTP/2. Use case: Chat, real-time sync.

---

## Full Comparison Table [CRITICAL]

| Dimension | REST | gRPC |
|---|---|---|
| **Data format** | JSON (text) | Protobuf (binary) |
| **Transport** | HTTP/1.1 (usually) | HTTP/2 (always) |
| **Schema** | Optional (OpenAPI) | Required (`.proto`) |
| **Code generation** | Optional | Built-in (`protoc`) |
| **Streaming** | ❌ Need SSE/WebSocket | ✅ 4 types built-in |
| **Browser support** | ✅ Native | ❌ Needs grpc-web proxy |
| **Human debuggable** | ✅ `curl`, DevTools | ❌ Need `grpcurl` |
| **Performance** | Slower | **2-10x faster** |
| **Caching** | ✅ HTTP cache, CDN | ❌ POST only, not cacheable |
| **API versioning** | URL/header (D12) | Field numbers (backward compatible) |

---

## Protobuf Versioning — Built-in Backward Compatibility [IMPORTANT]

```protobuf
// v1                          // v2 — added email
message User {                 message User {
  int32 id = 1;                  int32 id = 1;
  string name = 2;               string name = 2;
}                                string email = 3;  // NEW
                               }
```

- Old client + new server: client ignores unknown field 3. ✅
- New client + old server: field 3 returns default value (empty string). ✅

**Rules:**
- ✅ Add new fields (new number) — always safe
- ✅ Remove fields — old data ignored
- ❌ Never reuse a field number
- ❌ Never change a field's type

---

## When to Use Which [CRITICAL]

### Use REST:
| Scenario | Why |
|---|---|
| Browser-facing API | Browsers natively support fetch/XHR |
| Public API (third-party devs) | Anyone can `curl`. JSON universal. |
| Need HTTP caching / CDN | REST cacheable (D34). gRPC not. |
| Simple CRUD | REST resource model is natural |

### Use gRPC:
| Scenario | Why |
|---|---|
| Service-to-service (microservices) | No browser. Binary speed matters. |
| High throughput (10K+ calls/sec) | 2-10x faster serialization |
| Strict contracts needed | Compile-time type checking |
| Streaming required | Built-in 4 types |
| Polyglot services | One `.proto` → code for all languages |

### Common Production Pattern

```
Browser ── REST (JSON) ──→ API Gateway ── gRPC (Protobuf) ──→ Internal Services
           (human-friendly)                (machine-fast)
```

REST on the outside (browsers, third-party). gRPC on the inside (service-to-service).

---

## Common Mistakes

1. **gRPC for browser-facing APIs.** Browsers don't natively support gRPC. Need grpc-web proxy — complexity for no benefit.

2. **REST for high-volume internal calls.** At 50K calls/sec, JSON serialization overhead is measurable. gRPC saves CPU and latency.

3. **Reusing protobuf field numbers.** Reuse a deleted field number → data silently misinterpreted. No error thrown.

4. **Forgetting gRPC isn't HTTP-cacheable.** All gRPC calls are HTTP/2 POST. CDNs can't cache POST. Need Redis/app-level cache (D26, D32).

---

## Quiz + Answers

**Q1.** Orders Service calls Users Service 30K times/sec. Currently REST + JSON. Switch to gRPC?

**A1.** Yes. Benefits: (1) Protobuf ~50-80% smaller → less bandwidth at 30K/sec. (2) Binary serialization 2-10x faster → saves CPU. (3) HTTP/2 multiplexing → thousands of RPCs on one connection. (4) `.proto` catches type mismatches at compile time. You lose: `curl` debugging (need `grpcurl`), HTTP caching (need Redis), and team needs to learn protobuf. Worth it at 30K/sec.

---

**Q2.** Product team says "use gRPC for our public third-party API." Your response?

**A2.** Push back. Use REST + JSON. (1) Every developer knows REST — `curl`, Postman, any HTTP library. gRPC requires protobuf compiler, generated clients, gRPC libraries — high barrier. (2) JSON is human-readable for debugging. Protobuf is binary. (3) REST cacheable by CDNs. Keep gRPC for internal; REST for external.

---

**Q3.** Search service finds results incrementally: first 10 in 50ms, next 100 in 500ms, final 1000 in 3s. REST makes client wait 3s for all results. gRPC improvement?

**A3.** Use **server streaming**: `rpc Search(SearchRequest) returns (stream SearchResult)`. Server streams results as found. Client gets first 10 in 50ms, starts rendering immediately. User sees results appearing progressively instead of blank screen for 3s. Similar to SSE (D11) but binary protobuf over HTTP/2 — more efficient, type-safe.
