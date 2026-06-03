# D14 — Revision Day 2: Full Phase 1 Sweep (D01–D05)

**Date completed:** 2026-06-03
**Covered:** D01 DNS/TCP/NAT, D02 Sockets/Concurrency, D03 HTTP versions, D04 TLS deep dive, D05 REST
**Pre-quiz clarifications:** ECDHE's role in encryption, Hashing vs Encryption vs Signatures, TLS transcript mechanics

---

## Pre-Quiz Clarifications (follow-up questions before quiz)

### Does ECDHE decrypt HTTP data?

ECDHE does not encrypt or decrypt anything directly. It produces a shared secret. That secret is fed into a Key Derivation Function to produce an AES symmetric key. AES is what actually encrypts and decrypts all HTTP frames.

```
ECDHE runs once during handshake
        ↓
Both sides compute same shared secret (32-byte number)
        ↓
KDF (Key Derivation Function) turns it into an AES-256 key
        ↓
AES-256 encrypts/decrypts every HTTP frame
```

**AES** = Advanced Encryption Standard — the symmetric cipher doing the actual work.
**KDF** = Key Derivation Function — converts raw secret into a proper cryptographic key.

---

### Hashing vs Encryption vs Signature

| | Hashing | Encryption | Signature |
|---|---|---|---|
| Reversible? | No — one-way | Yes — with key | Partially — verifiable, not reversible |
| Key involved? | No | Yes | Yes (private key creates, public key verifies) |
| Purpose | Fingerprint / integrity check | Hide data from others | Prove who created something |
| Example | SHA-256 | AES-256, ChaCha20 | ECDSA, RSA |

**Hashing:** input → fixed-length output, cannot go back.
- Same input always = same hash. Tiny change in input = completely different hash (avalanche effect).
- Use cases: password storage (store hash not plaintext), file integrity checks, inside signatures.

**Encryption:** input + key → scrambled output. With the right key, unscramble it.
- Symmetric (AES): same key encrypts and decrypts. Fast.
- Asymmetric (RSA): public key encrypts, private key decrypts. Slow. Used for key exchange, not bulk data.

**Digital Signature:** hash of message + encrypt that hash with private key.

```
SIGNING:
  1. transcript = all handshake messages so far
  2. digest     = SHA-256(transcript)             ← hash it
  3. signature  = privateKey.sign(digest)         ← lock with private key
  4. send: transcript + signature

VERIFYING:
  1. digest_A = SHA-256(transcript — my own local copy)
  2. digest_B = publicKey.verify(signature)       ← unlock with public key
  3. digest_A == digest_B → valid ✓
```

**One-line answer:** Hashing = fingerprint (irreversible, no key). Encryption = lockbox (key required, reversible). Signature = proof of authorship (private key creates, public key verifies — built from hashing + asymmetric encryption combined).

---

### How all three appear in TLS

```
Handshake:
  Server signs transcript     → hashing + private key (signature)
  Client verifies signature   → public key from certificate

Data transfer:
  Every HTTP frame encrypted  → AES-256 (symmetric encryption)
  Every frame has HMAC        → keyed hash for integrity
```

**HMAC** = Hash-based Message Authentication Code — a keyed hash proving data wasn't tampered with AND that the sender holds the correct key.

---

### What is the TLS transcript exactly?

Both sides independently build a running log of every handshake message exchanged. Nobody sends the transcript — both sides build their own identical copy as messages flow.

```
TLS 1.3 handshake messages, in order:
  Step 1 → Client sends: ClientHello
  Step 2 → Server sends: ServerHello
  Step 3 → Server sends: EncryptedExtensions
  Step 4 → Server sends: Certificate  ← contains public key [C]
  ─────────────────────────────────────────
           SERVER SIGNS transcript of steps 1–4 HERE
  Step 5 → Server sends: CertificateVerify  ← this IS the signature
  Step 6 → Server sends: Finished
  Step 7 → Client sends: Finished
```

Server hashes steps 1–4, signs the hash, sends only the signature in step 5.
Client builds its own identical copy of steps 1–4, hashes it, decrypts signature, compares.

```
CLIENT                              SERVER
  │── ClientHello ────────────────▶ │  both append to transcript
  │◀─ ServerHello ─────────────────  │  both append to transcript
  │◀─ EncryptedExtensions ──────────  │  both append to transcript
  │◀─ Certificate ──────────────────  │  both append to transcript
  │                                   │  server: digest = SHA-256(transcript)
  │                                   │  server: sig = sign(digest, privateKey)
  │◀─ CertificateVerify (sig only) ── │
  │                                   │
  │  client: digest_A = SHA-256(my transcript copy)
  │  client: digest_B = verify(sig, publicKey[C])
  │  digest_A == digest_B ✓
```

**Why the transcript is signed, not just the cert:** The signature covers the actual ECDHE key shares exchanged in this specific handshake. A man-in-the-middle who swaps key shares would break the signature — their swapped keys produce a different transcript hash.

---

### Are ECDHE key shares in the transcript? Can attacker derive the secret?

Yes — the public parts of ECDHE are in the transcript and transmitted in plaintext. That's fine.

```
ClientHello:  contains public_A = curve_math(secret_A)   ← transmitted
ServerHello:  contains public_B = curve_math(secret_B)   ← transmitted
              secret_A                                    ← NEVER transmitted
              secret_B                                    ← NEVER transmitted
```

Attacker sees `public_A` and `public_B`. To compute the shared secret they need `secret_A` or `secret_B`. Neither was ever sent. The math makes computing secret from public mathematically impossible — that's the entire security foundation of ECDHE.

---

## Phase 1 Concept Anchors

### D01 Key Points

**DNS hierarchy:**
```
Browser cache → OS cache → Recursive Resolver → Root NS → TLD NS → Authoritative NS
```
Root only knows which server manages each TLD. TLD only knows which authoritative server manages each domain. Only authoritative knows the actual IP.

**NAT flow:**
```
Laptop (192.168.1.5:49832) → Router rewrites to public_IP:49832 → Internet → Server
Router saves: 192.168.1.5:49832 ↔ public_IP:49832
Response: Server → public_IP:49832 → Router looks up table → 192.168.1.5:49832 → Laptop
```

**TTL production tip:** Lower TTL to 300s 24 hours BEFORE changing IPs. If you change the IP first, old caches hold the stale IP for however long the original TTL was.

---

### D02 Key Points

**Socket distinction:**
```
Listening socket  → ONE, 0.0.0.0:port, never carries data, births new sockets
Connection socket → ONE PER CLIENT, actual TCP connection

1000 clients = 1001 sockets (1 listening + 1000 connection)
```

**4-tuple:** TCP identifies a connection by `(src_IP, src_port, dst_IP, dst_port)`. Multiple clients on same destination port are distinguished by their source IP/port.

**Why stateless enables horizontal scaling:** Stateful (session in-memory) forces requests to same server. Stateless (token in every request) means any server handles any request.

---

### D03 Key Points

**The critical stream distinction:**

| HTTP/2 "streams" | HTTP/3 QUIC streams |
|---|---|
| Logical labels (just a number in the frame header) on ONE TCP pipe | Genuinely independent transport streams |
| Packet loss on stream 3 freezes ALL streams | Packet loss on stream 3 stalls only stream 3 |
| TCP is in kernel — app layer cannot work around it | QUIC handles per-stream recovery in userspace |

**HPACK vs QPACK:**
- HPACK: shared table updated in packet-arrival order → order-dependent → HOL block bleeds into header decompression
- QPACK: same shared table, but updates flow on a separate encoder stream → each request stream decodes independently → no HOL block

**HTTP/3 discovery:** First connection is always HTTP/2 (TCP). Server returns `Alt-Svc: h3=":443"; ma=86400`. Browser caches it. Next visit tries QUIC. UDP/443 blocked → falls back to HTTP/2.

---

### D04 Key Points

**Two public keys from the server — the most confused concept in TLS:**

| | Certificate public key [C] | ECDHE key share [B] |
|---|---|---|
| Lifespan | Months/years — in the certificate | Per connection — thrown away after |
| Purpose | Identity: proves server owns the cert | Key exchange: both sides derive shared secret |
| Used for data? | Never | Never — only to produce the secret |

**Forward secrecy:** ECDHE generates and deletes ephemeral keys per connection. Stolen private key = cannot decrypt past traffic (ephemeral keys are gone). Can only impersonate for future connections.

**Certificate expiry is client-side:** Server keeps running. Only new clients after expiry are blocked. Existing connections (handshake already done) are unaffected.

**mTLS** = both sides present certificates. Used for service-to-service (microservices, VPNs). Never for browser users.

---

### D05 Key Points

**Safe vs idempotent:**
- Safe = does not modify server state (GET only)
- Idempotent = calling N times = same result as once (GET, PUT, DELETE)
- DELETE is idempotent but not safe — modifies state, but repeating it has same end result

**Method table:**

| Method | Safe? | Idempotent? |
|---|---|---|
| GET | Yes | Yes |
| POST | No | No |
| PUT | No | Yes |
| PATCH | No | Sometimes |
| DELETE | No | Yes |

**401 vs 403:**
- 401 = "I don't know who you are" (no token, expired token, not logged in)
- 403 = "I know who you are, you just can't do this"

---

## Full Phase 1 Quiz — 12 Questions & Answers

### D01

**Q1. DNS walkthrough for `https://news.ycombinator.com` — nothing cached.**

```
1. Browser DNS cache         → miss
2. OS DNS cache              → miss
3. Recursive Resolver        → miss → starts hunting
4. Asks Root NS: "Who manages .com?"
   Root returns: IP of .com TLD server
5. Asks .com TLD Server: "Who manages ycombinator.com?"
   TLD returns: ns1.ycombinator.com (authoritative server address)
6. Asks ns1.ycombinator.com: "What is news.ycombinator.com?"
   Auth returns: 209.216.230.240  TTL: 300s
7. Answer flows back. Everyone caches for 300s.
   Browser proceeds to TCP.
```

---

**Q2. 20% users still hitting old server 3 hours after DNS change. Root cause?**

Root cause: DNS TTL on the old A record was high (e.g. 10800 = 3 hours). Recursive resolvers worldwide had the old IP cached. Changing the source record doesn't invalidate existing caches.

Fix — do this 24 hours before any IP change:
```
Step 1 (24h before): Lower TTL 10800 → 300. Wait 24h for all old caches to expire.
Step 2 (deploy):     Change A record to new IP. Now worst-case propagation = 5 minutes.
Step 3 (after):      Raise TTL back to normal.
```
Changing IP before lowering TTL = old TTL caches keep routing to the dead server for hours.

---

**Q3. NAT flow for `192.168.1.8:49832 → 93.184.216.34:443`**

```
OUTGOING at router:
  Packet: 192.168.1.8:49832 → 93.184.216.34:443
  Router rewrites source: public_IP:49832 → 93.184.216.34:443
  Saves in NAT table: 192.168.1.8:49832 ↔ public_IP:49832

INCOMING (server response):
  Packet: 93.184.216.34:443 → public_IP:49832
  Router checks NAT table → 192.168.1.8:49832
  Rewrites destination, delivers to laptop ✓
```

Port `:49832` distinguishes this device from all other devices sharing the same public IP.

---

### D02

**Q4. Thread pool = 50, DB call = 800ms, 60 concurrent users. 51st request? Does async help?**

With threads: 50 threads all blocked waiting on DB (BLOCKED state, doing nothing). 51st request → no thread available → queued or rejected with 503.

Async helps because one thread handles all 60 connections. "Waiting on DB" = event loop sent query and moved on to the next request. OS notifies via epoll when DB responds.

```
Thread model:   50 threads all stuck waiting 800ms — 51st queued/rejected
Async model:    1 thread sends 60 DB queries, handles responses as they arrive
                no blocking, all 60 in-flight simultaneously
```

Caveat: async only helps for I/O-bound work (DB, network, file reads). CPU-heavy work blocks the single thread and freezes all connections.

---

**Q5. Session on Server 1's memory. Next request hits Server 2. Two fixes.**

User sees: logged out, 401, or "not authenticated" — Server 2 has no record of this session.

Fix 1 — Sticky sessions: load balancer routes same user to same server (by IP hash or cookie). Problem: server goes down → sessions lost; limits even load distribution.

Fix 2 — Shared Redis session store: all servers read/write sessions from a central Redis instance. Any server handles any request. True stateless scaling.

**Redis** = Remote Dictionary Server — in-memory key-value store, microsecond reads. Standard tool for shared sessions.

---

### D03

**Q6. HTTP/2 API slower than HTTP/1.1 on 10% packet loss network. Is the developer wrong?**

He is right. HTTP/2 uses ONE TCP connection. TCP in the kernel guarantees in-order delivery. When a packet is lost, the kernel buffers everything after the gap and delivers nothing until the lost packet is retransmitted. All streams on that one connection freeze — even streams whose own data arrived fine.

HTTP/1.1 with 6 connections: packet loss on connection 1 only stalls connection 1. Connections 2–6 keep flowing independently.

On lossy networks: HTTP/2's one connection = one loss point for all streams. Six HTTP/1.1 connections = six independent loss points. HTTP/3 fixes this — QUIC streams are independent at transport layer, so loss only stalls the affected stream.

---

**Q7. Wi-Fi → 4G mid-download. HTTP/2 vs HTTP/3.**

HTTP/2: TCP identifies connection by 4-tuple `(src_IP, src_port, dst_IP, dst_port)`. IP changes → 4-tuple changes → TCP sees a brand new connection. Old connection is dead. Download dies, must restart.

HTTP/3: QUIC embeds a Connection ID (CID) in every UDP packet. CID is a random identifier that has nothing to do with IP.
```
Wi-Fi:   src_IP = 192.168.1.5,  CID = 0xAB12CD34
4G:      src_IP = 10.0.0.2,     CID = 0xAB12CD34  ← same CID
Server: "I know this CID — same connection, IP changed, fine."
Download continues from exactly where it paused. ✓
```

---

### D04

**Q8. Two public keys from server in TLS 1.3. Name them, purpose, why both needed.**

Certificate public key [C]: long-lived (months/years), in the certificate, used for identity — client verifies server owns the cert via signature check.

ECDHE key share [B]: ephemeral (fresh per connection, thrown away after), used for key exchange — both sides compute the same shared secret using their private numbers and each other's public share.

Why not use [C] for everything?

1. Speed: asymmetric encryption is ~100x slower than AES. Cannot encrypt gigabytes of HTTP traffic with it.
2. Forward secrecy: [C]'s private key is long-lived. If an attacker records traffic today and steals the key later, they could decrypt everything. ECDHE's ephemeral secret is thrown away per connection — past traffic stays safe forever.

---

**Q9. Attacker records 6 months of traffic, then steals TLS private key. Can they decrypt?**

No. Forward secrecy prevents this.

The private key was used exactly once per connection — to sign the handshake transcript (prove identity). It was never used to derive the AES key or encrypt any data.

The AES key came from ECDHE ephemeral values: `secret_A` (client) + `secret_B` (server). Both were generated fresh per connection and deleted immediately after. They are gone. The private key cannot help recompute them.

What the stolen private key does allow: impersonating the server for new future connections. It cannot touch past recorded traffic.

---

### D05

**Q10. POST /payments — network drops after server processes, before response arrives. Client retries. Fix.**

Without handling: POST is not idempotent. Server already charged the user. Retry = server charges again. Double payment.

Fix — Idempotency Key pattern:

Client: generate a UUID before sending. Include it in every attempt with the same request.
```
POST /payments
Idempotency-Key: f47ac10b-58cc-4372-a567-0e02b2c3d479
```

Server:
```
Check: have I seen key f47ac10b... before?
  NO  → process payment → store result keyed by f47ac10b... → return result
  YES → return stored result, do not process again
```

On retry: same UUID → server returns cached result → no second charge. This is exactly how Stripe's API works.

**UUID** = Universally Unique Identifier — a random 128-bit ID statistically guaranteed to never collide.

---

**Q11. Method and status code for each scenario.**

| Scenario | Method | Status |
|---|---|---|
| a) Fetch user's profile | GET | 200 OK |
| b) Create a new order | POST | 201 Created |
| c) Change only a user's email | PATCH | 200 OK |
| d) Delete a post (successfully) | DELETE | 204 No Content |
| e) Logged-in user deletes someone else's post | DELETE | 403 Forbidden |

(e) is 403, not 401. 401 = server doesn't know who you are. 403 = server knows exactly who you are, you just don't have permission for this resource.

---

**Q12. "REST is just about using JSON and HTTP." Accurate?**

Not accurate on both counts. JSON is not required — REST says nothing about data format. You can return XML, plain text, or anything.

REST is a set of architectural constraints for using HTTP predictably:
1. HTTP verbs as actions: GET = read, POST = create, PUT = replace, DELETE = delete
2. URLs identify resources (nouns, not verbs)
3. Status codes used honestly
4. Stateless — each request carries everything needed, no server-side sessions

**The constraint with real production consequences — idempotency:**

Knowing which methods are idempotent (GET, PUT, DELETE) vs not (POST) determines retry safety across the entire system. Getting this wrong causes double charges, duplicate records, data corruption. That's not stylistic — it's a production contract that distributed systems depend on.

---

## Key Terms — Full Phase 1 Glossary

| Term | Meaning |
|---|---|
| Recursive Resolver | DNS server that hunts on your behalf (8.8.8.8 or ISP) |
| Root Name Server | Top of DNS hierarchy — knows only which server manages each TLD |
| Authoritative NS | Final answer holder — knows actual IPs for a domain |
| TTL | Time To Live — how long a DNS answer is cached before asking again |
| NAT | Network Address Translation — maps private IPs to one public IP |
| 4-tuple | (src_IP, src_port, dst_IP, dst_port) — uniquely identifies a TCP connection |
| Listening socket | Bound to a port; births new sockets for each client |
| Connection socket | Actual TCP connection to one specific client |
| epoll | Linux OS API: "tell me which of these sockets have data ready right now" |
| HOL blocking | Head-of-Line blocking — slow item at front blocks everything behind it |
| HPACK | HTTP/2 header compression — shared ordered table, order-dependent |
| QPACK | HTTP/3 header compression — same table, independently decodable via control streams |
| CID | Connection ID — QUIC's connection identifier; survives IP changes |
| Forward Secrecy | Past traffic stays safe even if long-term private key is stolen later |
| ECDHE | Elliptic Curve Diffie-Hellman Ephemeral — per-connection key exchange in TLS 1.3 |
| Symmetric encryption | Single key encrypts and decrypts (AES) — fast, used for bulk data |
| Asymmetric encryption | Key pair: public encrypts, private decrypts — slow, used for key exchange |
| Hashing | One-way function — input → fixed digest, irreversible, no key (SHA-256) |
| Digital Signature | Hash of data encrypted with private key — proves authorship and integrity |
| Transcript | Running log of all TLS handshake messages — both sides build independently |
| KDF | Key Derivation Function — converts ECDHE shared secret into a usable AES key |
| mTLS | Mutual TLS — both sides present certificates; used for service-to-service |
| Idempotent | Calling N times = same result as calling once |
| Safe | Does not modify server state (GET only) |
| UUID | Universally Unique Identifier — random 128-bit ID used as idempotency key |
| Redis | In-memory key-value store — standard tool for shared session storage |
