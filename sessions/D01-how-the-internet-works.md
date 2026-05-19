# D01 — How the Internet Works

**Date completed:** 2026-05-19
**Time:** ~1 hour + deep Q&A

---

## The Full Journey — What Happens When You Type a URL

```
You type: https://www.google.com

1. Browser checks DNS cache
2. OS checks DNS cache
3. DNS Lookup (recursive resolver → root → TLD → authoritative)
4. TCP 3-way handshake
5. TLS handshake (encryption setup)
6. HTTP Request sent
7. Server processes → HTTP Response
8. Browser renders page
```

---

## IP Addresses

Every device on the internet has an IP address — like a home address for machines.

**IPv4** — 4 numbers separated by dots, each 0–255:
```
192.168.1.1
```
Only ~4.3 billion possible. We ran out. That's why NAT and IPv6 exist.

**IPv6** — 128-bit address, essentially unlimited:
```
2607:f8b0:4004:0800::200e
```

### Public vs Private IP

| Type | Example | Where it works |
|---|---|---|
| Private | 192.168.1.5 | Only inside your home/office network |
| Public | 203.45.12.88 | On the real internet |

**Private IP ranges** (reserved, never routed on internet):
```
10.0.0.0    – 10.255.255.255    (large orgs)
172.16.0.0  – 172.31.255.255   (medium orgs)
192.168.0.0 – 192.168.255.255  (home networks)
```

Your router assigns private IPs to your devices via **DHCP** (Dynamic Host Configuration Protocol — automatic IP assignment). Your ISP assigns one public IP to your router.

---

## NAT — Network Address Translation

**Problem:** Multiple devices at home, only one public IP from ISP.
**Solution:** Router translates between private and public IPs.

### How it works

```
Your laptop (192.168.1.5) wants to reach Google (142.250.80.4)

OUTGOING:
Laptop sends packet:
  From: 192.168.1.5:54231  →  To: 142.250.80.4:443

Router intercepts, rewrites source, saves mapping:
  From: 203.45.12.88:54231  →  To: 142.250.80.4:443

NAT TABLE (router's memory):
  192.168.1.5:54231  ←→  203.45.12.88:54231

INCOMING (Google's response):
  Arrives at: 203.45.12.88:54231
  Router checks NAT table → maps to 192.168.1.5:54231
  Forwards to your laptop
```

**Key insight:** The port number is what distinguishes your laptop from your phone — both share one public IP but use different ports.

**Why local servers aren't reachable from internet:** NAT table only has entries for outgoing connections. Incoming requests from outside have no mapping. Solution: Port Forwarding (you manually tell router "incoming :8080 → 192.168.1.5:8080").

---

## Ports — Apartment Numbers for IP Addresses

IP address = building address. Port = apartment number.

One server runs many services on different ports simultaneously.

| Port | Service |
|---|---|
| 80 | HTTP (unencrypted web) |
| 443 | HTTPS (encrypted web) |
| 22 | SSH (remote terminal) |
| 53 | DNS |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 3306 | MySQL |

**Default ports:** Browsers automatically use port 443 for `https://` and 80 for `http://`. `https://google.com` is actually `https://google.com:443` — browser hides the default.

If your app runs on port 3000, you must specify it: `http://localhost:3000`

Ports 0–1023 are "well-known ports" — need root/admin to listen on them. That's why dev servers use 3000, 8000, 8080.

---

## DNS — Domain Name System

Translates human-readable domain names → IP addresses.

**The 4 players:**

| Player | Role |
|---|---|
| Recursive Resolver | Does the hunting on your behalf (your ISP or 8.8.8.8 / 1.1.1.1) |
| Root Name Server | Knows which server manages each TLD (.com, .org, .in) |
| TLD Name Server | Knows which server manages each domain within that TLD |
| Authoritative Name Server | Knows the actual IP — final answer |

### Full DNS Lookup for `www.google.com`

```
Step 1 — Browser cache: "Do I know this already?" → NO

Step 2 — OS cache + /etc/hosts: "Do YOU know it?" → NO

Step 3 — Recursive Resolver (your ISP or 8.8.8.8):
  "Is it cached?" → NO → start hunting

Step 4 — Asks Root Server:
  Root: "I don't know google.com's IP.
         But .com is managed by 192.5.6.30"

Step 5 — Asks .com TLD Server (192.5.6.30):
  TLD: "I don't know the IP.
        But google.com's authoritative server is ns1.google.com"

Step 6 — Asks ns1.google.com (Google's authoritative server):
  Auth: "www.google.com → 142.250.80.4  TTL: 300s"

Step 7 — Answer flows back. Everyone caches it for 300s.
```

### Root Servers
- 13 addresses (A through M), hardcoded in every DNS resolver
- Not 13 machines — 13 addresses, each backed by hundreds of servers worldwide
- Use **anycast** — your query goes to nearest physical server automatically
- Know ONLY which TLD server manages each TLD. Nothing else.

### TTL (Time To Live)
Every DNS answer has a TTL in seconds. After it expires, resolver asks again.

```
Production tip: Before changing your server IP:
1. Lower TTL to 300 (5 min) and wait 24 hours (old TTL to expire)
2. THEN change the IP
3. Propagation now takes only 5 min instead of 24 hours
```

### DNS Record Types

| Record | Does |
|---|---|
| A | domain → IPv4 address |
| AAAA | domain → IPv6 address |
| CNAME | domain → another domain (alias) |
| MX | who handles email for this domain |
| TXT | arbitrary text (used for domain verification, SPF, DKIM) |

**Why CNAME matters:** Vercel/Netlify tell you to set a CNAME to `xyz.vercel.app`. Their IP can change anytime — your CNAME follows automatically because it points to their domain, not a raw IP.

---

## TCP — Transmission Control Protocol

Guarantees data arrives **completely, in order, uncorrupted**.

Data travels as **packets** (~1.5KB each). A 1MB file = ~700 packets, potentially taking different routes, arriving out of order. TCP reassembles them correctly.

### 3-Way Handshake

Before any data flows, both sides sync their **sequence numbers** (counters that track every byte sent):

```
Client                              Server
  │                                    │
  │──── SYN (seq=3271) ──────────────▶ │  "I want to connect, I'll count from 3271"
  │                                    │
  │ ◀── SYN-ACK (seq=8642, ack=3272) ─│  "Ok, I'll count from 8642, send me 3272 next"
  │                                    │
  │──── ACK (ack=8643) ──────────────▶ │  "Got it. Your turn starts at 8642."
  │                                    │
  │──── HTTP Request ────────────────▶ │  NOW data flows
```

**Sequence numbers:** How TCP guarantees order. Every byte is numbered. If packet with seq=2000 goes missing, receiver keeps sending ACK=2000 until sender retransmits it.

**Random starting seq numbers:** Prevents confusion if old packets from a previous connection arrive late.

### Latency — The Physics Problem

```
Speed of light in fiber ≈ 200,000 km/s
India → US distance    ≈ 13,000 km
Round trip time        ≈ 130ms (minimum, physics)
Real world             ≈ 150–250ms (router delays added)
```

This means just the TCP handshake costs ~200ms from India to US — before a single byte of your request is sent. **This is why CDNs exist.**

CDN terminates TCP connection locally (e.g., Mumbai for Indian users), then has a persistent warm connection to your origin server. TCP handshake: India ↔ Mumbai = ~10ms instead of 200ms.

---

## TLS — Transport Layer Security (HTTPS)

Three jobs:
1. **Authentication** — prove server is who it claims to be
2. **Key Exchange** — agree on encryption keys without sending them in plaintext
3. **Encryption** — scramble all data so nobody can read it in transit

### Certificates & Certificate Authorities (CAs)

**CA (Certificate Authority):** A company that browsers trust by default (~150 built into Chrome/Firefox/OS). Examples: DigiCert, Let's Encrypt, Comodo.

**How it works:**
```
Google → goes to DigiCert → "I own google.com, give me a certificate"
DigiCert → verifies ownership (DNS checks etc.)
DigiCert → signs a certificate:
  Domain:     google.com
  Public Key: [Google's public key]
  Expires:    2024-12-31
  Signed by:  DigiCert

When you connect to google.com:
  Server sends this certificate
  Browser checks DigiCert's signature → valid ✓
  Browser checks domain matches → google.com ✓
  Browser checks expiry → not expired ✓
  Conclusion: This IS google.com. Safe.
```

Nobody can fake this without DigiCert signing it — and DigiCert won't sign for a domain you don't own.

### TLS 1.2 vs TLS 1.3

**TLS 1.2 — 2 round trips:**
```
Round Trip 1:
  Client: "I support TLS 1.2, here are my cipher options"
  Server: "Use AES-256. Here's my certificate."

Round Trip 2:
  Client: "Here's the pre-master secret (encrypted with your public key)"
  Server: "Confirmed. Switching to encrypted."

→ Data flows (2 round trips = ~400ms India→US)
```

**TLS 1.3 — 1 round trip:**
```
Round Trip 1:
  Client: "I support TLS 1.3. I'm guessing you'll pick X25519 cipher.
           Here's my key share for that cipher already."
  Server: "Correct guess. Here's my certificate + key share.
           Already encrypted from here."

→ Data flows (1 round trip = ~200ms India→US)
```

Client guesses the cipher upfront (TLS 1.3 removed weak ciphers, fewer options = easier to guess right). Saves one full round trip.

**0-RTT (Zero Round Trip):** On reconnect, client sends request with zero handshake using a saved session ticket. Only safe for GET requests — vulnerable to replay attacks for writes.

### Full Timeline Comparison

```
TLS 1.2, no CDN (India → US):
  TCP handshake:     200ms
  TLS Round Trip 1:  200ms
  TLS Round Trip 2:  200ms
  HTTP Request:      200ms
  Total: 800ms ──────────────────────── user waits 0.8s

TLS 1.3, no CDN (India → US):
  TCP handshake:     200ms
  TLS (1 round trip): 200ms
  HTTP Request:      200ms
  Total: 600ms ──────────────────────── saved 200ms

TLS 1.3 + CDN in Mumbai:
  TCP + TLS to Mumbai: ~20ms
  CDN serves cache:    ~10ms
  Total: ~30ms ──────────────────────── 27x faster
```

---

## HTTP — The Actual Request/Response

After TCP + TLS, the actual work:

**Request:**
```http
GET /search?q=hello HTTP/2
Host: www.google.com
Accept: text/html
User-Agent: Chrome/120
```

**Response:**
```http
HTTP/2 200 OK
Content-Type: text/html
Content-Length: 14263

<!DOCTYPE html>...
```

**Status codes:**
```
2xx → Success        200 OK, 201 Created, 204 No Content
3xx → Redirect       301 Permanent, 302 Temporary
4xx → Your fault     400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found
5xx → Server fault   500 Internal Error, 502 Bad Gateway, 503 Service Unavailable
```

**503 vs 500:**
- 500 = server crashed / threw an exception
- 503 = server is up but overloaded or down for maintenance

---

## Terminal Commands — See It Yourself

```bash
# DNS lookup — what IP does a domain resolve to?
dig google.com

# Watch the full DNS hierarchy (root → TLD → authoritative)
dig +trace www.google.com

# See every router hop between you and Google
traceroute google.com

# See full HTTP conversation (TCP connect, TLS, request, response)
curl -v https://google.com 2>&1 | head -60

# Check DNS TTL value
dig google.com | grep -A5 "ANSWER SECTION"
```

---

## Production Gotchas

| Situation | What goes wrong |
|---|---|
| DNS TTL too high | Changed server IP but users hit old server for hours |
| DNS TTL too low | More DNS queries, higher cost |
| No HTTPS | ISP can inject ads; browsers show "Not Secure" |
| TLS 1.2 still used | Slower handshake, weaker ciphers |
| Server far from users | High latency — physics, can't fix in code |
| No port forwarding | Local server unreachable from internet |
| Non-root port (80/443) blocked | Users can't reach app without specifying port |

**The #1 misconception:** Fast code ≠ fast website. A 1ms DB query means nothing if 400ms is spent on TCP+TLS handshake before your server starts. Network distance is often the bottleneck, not code.

---

---

## The Complete Flow — One Example Connecting Everything

You open your laptop at home and type `https://www.google.com` and hit Enter.

---

**Your laptop has IP `192.168.1.5` — a private IP assigned by your router via DHCP.**
Your router's public IP (given by ISP) is `203.45.12.88`.
The internet only knows `203.45.12.88`. Your `192.168.1.5` doesn't exist outside your home.

---

**Step 1 — Browser needs Google's IP. DNS lookup begins.**

```
Browser cache     → miss
OS cache          → miss
Recursive Resolver (8.8.8.8) → not cached, starts hunting

  Asks Root Name Server (198.41.0.4):
    Root: "I don't know. .com is managed by 192.5.6.30"

  Asks .com TLD Server (192.5.6.30):
    TLD: "I don't know the IP. google.com's auth server is ns1.google.com"

  Asks Google's Authoritative Server (ns1.google.com):
    Auth: "www.google.com → 142.250.80.4  TTL: 300s"

Answer flows back → everyone caches it for 300s
```

**Google's IP: `142.250.80.4`**

---

**Step 2 — TCP connection to `142.250.80.4:443`**

Port 443 because `https://` — browser adds it automatically as default.

Your laptop sends:
```
From: 192.168.1.5:54231  →  To: 142.250.80.4:443
```

**NAT happens at your router:**
- Rewrites source: `192.168.1.5:54231` → `203.45.12.88:54231`
- Saves in NAT table: `192.168.1.5:54231 ←→ 203.45.12.88:54231`
- Sends packet to internet

Packet crosses ~20 routers worldwide. Takes ~200ms (India → US, speed of light).

**TCP 3-way handshake:**
```
Laptop  →  SYN (seq=3271)              →  Google
Laptop  ←  SYN-ACK (seq=8642, ack=3272) ←  Google
Laptop  →  ACK (ack=8643)              →  Google
```
Both sides synced on sequence numbers. Connection established.

Google's replies come back to `203.45.12.88:54231`. Router checks NAT table → rewrites to `192.168.1.5:54231` → delivers to your laptop.

---

**Step 3 — TLS handshake. Encryption setup.**

```
Laptop  →  "TLS 1.3, here are cipher options + key share for X25519"
Google  ←  "Here's my certificate (signed by DigiCert, valid for google.com)"
```

Browser verifies certificate:
- DigiCert's signature valid? ✓
- Domain matches `google.com`? ✓
- Not expired? ✓

Both sides derive encryption keys. Everything from here is encrypted.
Your ISP, coffee shop router, every hop in between — all see garbage bytes.

---

**Step 4 — HTTP Request**

```http
GET / HTTP/2
Host: www.google.com
User-Agent: Chrome/120
```

Sent encrypted over the TCP connection.

---

**Step 5 — Google responds**

```http
HTTP/2 200 OK
Content-Type: text/html

<!DOCTYPE html>...
```

Response broken into TCP packets, numbered by sequence numbers.
Arrive possibly out of order → TCP reassembles correctly → browser renders.

---

**Total time: ~600ms (India → US, TLS 1.3, no CDN)**

```
DNS lookup         ~100ms
TCP handshake      ~200ms   ← physics, can't avoid
TLS handshake      ~200ms   ← 1 round trip with TLS 1.3
HTTP request/resp  ~200ms
```

With a CDN in Mumbai → TCP + TLS terminate locally in ~20ms → page loads in ~100ms.

---

## Key Terms Glossary

| Term | Meaning |
|---|---|
| IP Address | Unique address for every device on a network |
| IPv4 / IPv6 | Two versions of IP address format (32-bit vs 128-bit) |
| NAT | Router technique to share one public IP across many private devices |
| DHCP | Protocol routers use to auto-assign private IPs to devices |
| Port | Number that identifies which service on a server you're talking to |
| DNS | System that translates domain names to IP addresses |
| TTL | How long a DNS answer is cached before asking again |
| Recursive Resolver | DNS server that does the hunting on your behalf |
| Root Server | Top of DNS hierarchy — knows all TLDs |
| TLD Server | Knows all domains within a TLD (.com, .org etc.) |
| Authoritative NS | Final answer holder — knows actual IPs for a domain |
| TCP | Protocol guaranteeing ordered, complete, reliable delivery |
| Packet | Small chunk data is broken into for internet transmission |
| Sequence Number | TCP's per-byte counter that enables ordering and loss detection |
| Handshake | Initial exchange between two parties before real communication |
| Latency | Time for a packet to travel from source to destination |
| TLS | Encryption layer on top of TCP — what makes HTTPS secure |
| CA | Trusted organization that signs certificates, vouching for domain ownership |
| Certificate | Server's identity document — domain + public key + CA signature |
| Anycast | One IP address, many physical servers — request goes to nearest |
| CDN | Network of servers at the edge, close to users |
| Port Forwarding | Manually telling router to route incoming traffic to a specific device |
| 0-RTT | TLS feature — send data with zero handshake on reconnect |

---

## Quiz (answered in session)

1. **A user in India says your site is slow. Server is in US. Code is fast. What's the cause?**
   Network latency (physics). TCP + TLS handshake ≈ 400–600ms before anything loads. Fix: put a CDN in India to terminate connections locally.

2. **You changed your server IP. Some users hit the old server 2 hours later. Why?**
   DNS TTL was high (e.g., 7200s = 2 hours). Old IP was cached. Fix: lower TTL to 300s 24h before migration, then change IP.

3. **What's the difference between 500 and 503?**
   500 = server threw an exception / crashed internally. 503 = server is alive but overloaded or in maintenance — can't handle requests right now.
