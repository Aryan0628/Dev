# D04 — TLS/HTTPS: Handshake, Certificates, Why It Matters

**Date completed:** 2026-05-22

---

## Why It Matters

HTTP is plaintext. Every byte — passwords, tokens, credit card numbers — travels readable by anyone in between: your router, your ISP, every network hop.

TLS (Transport Layer Security) encrypts the connection *before* any HTTP data is sent. HTTPS = HTTP running on top of TLS.

**Three guarantees TLS provides:**
1. **Confidentiality** — data is encrypted; eavesdroppers see gibberish
2. **Integrity** — data cannot be tampered with in transit without detection
3. **Authentication** — you're actually talking to the real server, not an impersonator

---

## Where TLS Sits in the Stack

```
Application Layer:  HTTP  (plaintext request/response)
                         ↕
Security Layer:     TLS   (encrypts everything above)
                         ↕
Transport Layer:    TCP   (reliable delivery)
                         ↕
Network Layer:      IP    (routing packets)
```

TLS is handled by a library (OpenSSL on Linux, Secure Transport on macOS) — **userspace code**, not the kernel. Your web server (Nginx etc.) links against this library. The OS handles TCP below it.

---

## What Is a Certificate?

A TLS certificate is a digital document that proves the server's identity — like a passport.

A certificate contains:
- The domain it's valid for (e.g. `stripe.com`)
- The server's **long-lived public key** [C]
- The CA's digital signature (proof the CA blessed it)
- An expiry date

**CA (Certificate Authority)** = a trusted company (DigiCert, Let's Encrypt, Comodo) that signs certificates. Your OS and browser come pre-loaded with ~150 CAs called the **root store**.

---

## Symmetric vs Asymmetric Encryption

**Asymmetric** — key pair: public key + private key
- Anything encrypted with the public key can only be decrypted with the private key
- Public key is shared with the world; private key never leaves the server

**Symmetric** — single shared key
- Same key encrypts and decrypts
- Much faster than asymmetric

TLS uses both:
- Asymmetric **during** the handshake — to safely establish a shared secret
- Symmetric **after** the handshake — for all actual data (fast)

---

## The TLS 1.3 Handshake

```
CLIENT                                      SERVER
  |                                            |
  |-- ClientHello ------------------------->   |
  |   "I support TLS 1.3"                     |
  |   "Here are cipher suites I know"         |
  |   client's ECDHE public key share [A]     |   ← key exchange key
  |                                            |
  |<-- ServerHello + Certificate -----------   |
  |   "Let's use TLS 1.3, cipher X"           |
  |   server's ECDHE public key share [B]     |   ← key exchange key
  |   certificate (contains public key [C])   |   ← identity key
  |   signature (made with private key)       |   ← proves server owns cert
  |                                            |
  | Client does three things:                  |
  |   1. Verify certificate using CA's key    |
  |      from root store → cert is genuine    |
  |   2. Verify signature using [C]           |
  |      → server actually owns the cert      |
  |   3. Compute shared secret: math(A, B)    |
  |      Server also computes: math(B, A)     |
  |      → both get same secret, nothing      |
  |        secret was ever transmitted        |
  |                                            |
  |<---- {Finished} (encrypted) -----------   |
  |----- {Finished} (encrypted) ---------->   |
  |                                            |
  |======= Encrypted HTTP starts ============ |
```

---

## Two Separate Public Keys from the Server

This is the most important thing to keep straight:

| | Certificate Public Key [C] | ECDHE Key Share [B] |
|---|---|---|
| What it is | Long-lived, lives in the certificate | Temporary, generated fresh per connection |
| Purpose | Prove server identity (authentication) | Derive the shared secret (key exchange) |
| Who uses it | Client verifies CA signature, then server's handshake signature | Both sides compute the shared secret |
| After handshake | Not used for data | Thrown away — forward secrecy |

---

## Where the Private Key Is Used

The server's private key **never leaves the server**. It is used exactly once — to prove the server owns the certificate.

```
SERVER (locally):
  signature = privateKey.sign(handshake_transcript)
  → sends only the SIGNATURE over the wire

CLIENT:
  valid = certificatePublicKey.verify(handshake_transcript, signature)
  → if valid, server is authenticated
```

What travels over the wire vs what stays on the server:

| What | Direction | Contains private key? |
|---|---|---|
| Certificate | Server → Client | No — contains public key [C] |
| ECDHE key share [B] | Server → Client | No — ephemeral public key |
| ECDHE key share [A] | Client → Server | No — ephemeral public key |
| Signature | Server → Client | No — output of private key math, not the key itself |

**Nothing with a private key ever leaves the machine that owns it.**

---

## Certificate Verification — Two Layers

**Layer 1 — Verify the certificate is genuine (using the CA)**

```
CLIENT:
  CA_public_key = rootStore["DigiCert"]   ← already on your OS
  CA_public_key.verify(certificate)       ← did DigiCert actually sign this?
  → if yes, certificate is genuine
```

**Layer 2 — Verify the server owns the certificate (using the handshake signature)**

```
CLIENT:
  server_public_key = certificate.publicKey   ← trusted because Layer 1 passed
  server_public_key.verify(handshake_signature)
  → if yes, server holds the private key → server is the real stripe.com
```

---

## Does the Server Verify the Client?

**No — by default TLS only authenticates the server, not the client.**

```
CLIENT                          SERVER
  |                                |
  |  "Here's your certificate"  <--|   server proves identity
  |                                |
  |  CLIENT verifies it            |   ← only this happens
  |  against root store            |
```

The server has no idea who the client is after the TLS handshake. Identity of the user comes later at the HTTP layer via login forms, session cookies, API keys, or JWTs (JSON Web Tokens — signed tokens carrying identity claims).

**mTLS (mutual TLS)** = the exception. The client also presents a certificate and the server verifies it. Used for machine-to-machine communication: microservices, VPNs, API clients. Not used on normal websites.

---

## Why ECDHE If DigiCert Already Verified the Server?

They solve **different problems**:

| Problem | Solved by |
|---|---|
| "Is this really stripe.com?" | Certificate + DigiCert verification |
| "What key do we use to encrypt actual data?" | ECDHE |

You can't just encrypt data with the certificate's public key because:
1. **Speed** — asymmetric encryption is ~100× slower than symmetric. Cannot encrypt gigabytes of traffic with it.
2. **Forward Secrecy** — the certificate private key is long-lived. If someone records your traffic today and steals the private key a year later, they can decrypt everything. ECDHE creates a one-time secret that is thrown away after the connection — stolen keys can never decrypt past traffic.

---

## How ECDHE Works

### Color mixing analogy

Mixing paint is easy. Unmixing is impossible.

```
PUBLIC color: Yellow (everyone knows)

CLIENT                              SERVER
picks secret: Red                   picks secret: Blue
mixes: Yellow + Red = Orange        mixes: Yellow + Blue = Green
sends Orange ─────────────────────>
             <───────────────────── sends Green

CLIENT                              SERVER
takes Green + Red = Brown           takes Orange + Blue = Brown
```

Both got Brown — the shared secret. Nobody sent Brown. Nobody sent Red or Blue. Attacker sees Yellow + Orange + Green and cannot compute Brown.

### What ECDHE actually does

Same idea, but with math on points on a curve (Elliptic Curve Diffie-Hellman):

- There's a public starting point on the curve (like Yellow)
- Each side picks a secret number and "multiplies" the point by it (easy math)
- Reversing it (finding the secret number from the result) = computationally impossible

```
CLIENT                               SERVER
secret_A = random number             secret_B = random number
public_A = point × secret_A          public_B = point × secret_B

sends public_A ─────────────────────>
              <───────────────────── sends public_B

shared = public_B × secret_A         shared = public_A × secret_B
       = point × secret_B × secret_A         = point × secret_A × secret_B
                         ↑ same result ↑
```

**Ephemeral** = the E in ECDHE. secret_A and secret_B are generated fresh each connection and deleted after. Past traffic stays safe even if someone steals the private key later.

---

## ECDHE Per Packet or Per Connection?

ECDHE runs **once per connection** — not per packet, not per request.

```
TCP connection opens
        ↓
TLS handshake runs ONCE
  → ECDHE happens here
  → shared secret derived
        ↓
All packets use that same secret
  → GET /dashboard    ← same secret
  → POST /payment     ← same secret
        ↓
TCP connection closes
  → secret thrown away forever
```

New TCP connection = new ECDHE = new shared secret.

---

## Same Client Requesting Two Routes at the Same Time

Depends on HTTP version (from D03):

**HTTP/1.1** — cannot multiplex → opens two separate TCP connections

```
Connection 1: TLS handshake → ECDHE → secret_1 → GET /dashboard
Connection 2: TLS handshake → ECDHE → secret_2 → GET /profile
```

**HTTP/2** — multiplexes both on one TCP connection → one ECDHE covers both

```
Connection: TLS handshake → ECDHE → secret_1
                ├── stream A → GET /dashboard  ← same secret
                └── stream B → GET /profile    ← same secret
```

| | HTTP/1.1 | HTTP/2 |
|---|---|---|
| Parallel requests | Multiple TCP connections | One TCP connection |
| ECDHE runs | Once per connection | Once total |
| Secrets | Different per connection | Same for all parallel requests |

---

## Code Demo — ECDHE in Terminal

```python
"""
ECDHE Demo — D04
Run: python3 sessions/ecdhe_demo.py
"""

# PART 1: Simple Diffie-Hellman with small numbers
# Same concept as ECDHE, verifiable by hand

p = 23   # public prime
g = 5    # public generator

client_secret = 6
server_secret = 15

client_public = pow(g, client_secret, p)   # 5^6 mod 23
server_public = pow(g, server_secret, p)   # 5^15 mod 23

# Both derive the same shared secret
client_shared = pow(server_public, client_secret, p)
server_shared = pow(client_public, server_secret, p)

assert client_shared == server_shared
# Attacker sees p, g, client_public, server_public — cannot compute the secret


# PART 2: Real ECDHE (X25519 — what TLS 1.3 actually uses)
from cryptography.hazmat.primitives.asymmetric.x25519 import X25519PrivateKey

client_private = X25519PrivateKey.generate()   # ephemeral — thrown away after
client_public  = client_private.public_key()

server_private = X25519PrivateKey.generate()   # ephemeral — thrown away after
server_public  = server_private.public_key()

client_shared = client_private.exchange(server_public)
server_shared = server_private.exchange(client_public)

assert client_shared == server_shared
# Run twice — different secret each time (ephemeral)
# This secret now encrypts all HTTP data for this connection
```

Run with: `python3 sessions/ecdhe_demo.py`

What to observe:
- Part 1: both sides compute the same number without transmitting it — verify the math manually
- Part 2: both 32-byte secrets are identical; run twice and they differ (ephemeral)

---

## Nginx TLS Config

```nginx
server {
    listen 443 ssl;                        # 443 = standard HTTPS port
    server_name api.example.com;

    ssl_certificate     /etc/ssl/example.com.crt;   # public cert (sent to clients)
    ssl_certificate_key /etc/ssl/example.com.key;   # private key (never leaves server)

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://app:3000;   # TLS terminated here; plain HTTP forwarded to app
    }
}
```

**TLS termination** = Nginx decrypts traffic and forwards plain HTTP internally. The app on port 3000 sees no TLS.

Get a free certificate:
```bash
certbot --nginx -d api.example.com
```

---

## Trade-offs

| Concern | Detail |
|---|---|
| Latency | TLS 1.3 adds ~1 RTT (Round Trip Time) to first connection; session resumption brings this to 0 |
| CPU cost | Negligible on modern hardware; TLS 1.3 uses efficient ciphers |
| Certificate management | Certs expire (90 days for Let's Encrypt); production outages from expired certs are common |
| Internal services | Services inside a private VPC (Virtual Private Cloud) often skip TLS and rely on network isolation — simpler, but weaker |

**RTT** = Round Trip Time — time for a packet to go to the server and come back.
**VPC** = Virtual Private Cloud — an isolated private network in the cloud.

When NOT to use HTTPS: almost never. Only loopback (`localhost`) or fully private networks with strict zero-trust guarantees. All public-facing endpoints must use HTTPS.

---

## Key Terms Glossary

| Term | Meaning |
|---|---|
| TLS | Transport Layer Security — encryption protocol; TLS 1.3 is current |
| HTTPS | HTTP running on top of TLS |
| CA | Certificate Authority — trusted company that signs certificates (DigiCert, Let's Encrypt) |
| Root store | Pre-loaded list of ~150 trusted CAs on your OS/browser |
| Certificate | Digital document proving server identity; contains domain, public key, CA signature, expiry |
| Asymmetric encryption | Public/private key pair; public key encrypts, only private key decrypts |
| Symmetric encryption | Single shared key; same key encrypts and decrypts; much faster |
| ECDHE | Elliptic Curve Diffie-Hellman Ephemeral — key exchange algorithm in TLS 1.3 |
| Ephemeral | Temporary; ECDHE keys are generated fresh per connection and thrown away |
| Forward Secrecy | Property where past traffic stays safe even if private key is stolen later |
| Handshake | Negotiation phase before any HTTP data flows; agrees on cipher, verifies identity, derives secret |
| TLS termination | When a proxy (Nginx) decrypts TLS and forwards plain HTTP to the app |
| mTLS | Mutual TLS — both client and server present certificates; used for machine-to-machine |
| JWT | JSON Web Token — signed token carrying identity claims; used for application-level auth |
| VPC | Virtual Private Cloud — isolated private network in the cloud |

---

## Quiz

**Q1.** A certificate expires tonight at midnight. What exactly breaks — and for whom?

**A:** The server keeps running fine — expiry is checked by the client, not the server. After midnight, any new client attempting to connect runs the certificate check, sees the expiry date has passed, and the browser blocks the connection with a certificate error. Existing connections established before midnight are unaffected — the handshake already completed. Only new connections fail.

---

**Q2.** The handshake is done. Your browser now sends `GET /dashboard`. Is this GET request visible to your ISP? Why or why not?

**A:** No. After the TLS handshake, all HTTP data — the path `/dashboard`, headers, cookies, body — is encrypted with the shared secret from ECDHE. Your ISP can only see: the destination IP + port 443, the SNI (Server Name Indication — the domain sent in plaintext during ClientHello so the server knows which certificate to present), and the volume/timing of traffic. The actual request content is fully encrypted and invisible.

---

**Q3.** A man-in-the-middle attacker intercepts your connection and presents a certificate for `stripe.com` that *they* generated. Why does this fail?

**A:** The attacker's certificate was not signed by any CA in your root store — they signed it themselves. When the client runs Layer 1 verification (`CA_public_key.verify(certificate)`), no trusted CA's key can verify that signature. The browser sees the check fail and blocks the connection with a certificate error. To succeed, the attacker would need to either compromise a trusted CA (rare but catastrophic when it happens) or trick the user into manually trusting their certificate. This is also why certificate pinning exists — apps hardcode which specific CA or cert to trust, so even a compromised CA cannot perform a man-in-the-middle attack.
