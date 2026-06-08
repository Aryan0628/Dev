# D34 — CDN Deep Dive

## One-line summary
A CDN caches copies of your content at geographically distributed edge nodes; requests from Cache-Control headers tell it what to cache, for how long, and for whom — and purge APIs let you invalidate that cache before TTL expires.

---

## Why It Matters

Without a CDN: every request (static assets, HTML, API responses) hits your origin server. A user in Mumbai talking to a US server adds ~200ms round-trip latency. A traffic spike takes down your origin.

With a CDN: users hit a nearby edge node. Origin only sees traffic on cache misses and for content that can't be cached.

---

## Real Case: Fastly Global Outage — June 8, 2021

- A customer changed a CDN config setting, triggering a dormant software bug introduced May 12.
- Within 1 minute, 85% of Fastly's network went offline.
- Sites down: Reddit, NY Times, Twitch, Stack Overflow, Gov.uk, Amazon, GitHub, Spotify.
- Duration: ~1 hour.
- Lesson: CDN is a single point of failure if you use only one provider. Multi-CDN with failover routing is required for critical systems.

---

## Terminology

| Term | Definition |
|---|---|
| **Origin server** | Your actual backend — source of truth |
| **Edge node** | CDN server at a PoP that caches and serves content |
| **PoP (Point of Presence)** | A CDN data center location |
| **Cache HIT** | Edge has the content cached — serves without going to origin |
| **Cache MISS** | Edge doesn't have it — fetches from origin, caches, serves |
| **Cache key** | The identifier used to look up a cached item (default: Host + URL + query string) |
| **ETag (Entity Tag)** | A fingerprint/hash of content that origin sends; used for conditional revalidation |

---

## How Requests Reach the Nearest Edge Node

**Anycast DNS [IMPORTANT]:**  
CDN provider advertises the same IP from multiple locations via BGP. Your TCP connection is automatically routed to the nearest PoP. The IP is identical everywhere; BGP routing does the geo-selection.

**GeoDNS [GOOD TO KNOW]:**  
DNS server returns different IPs based on client location. More explicit, more management overhead. Older approach.

---

## Cache-Control Headers [CRITICAL]

| Directive | Who respects it | Meaning |
|---|---|---|
| `max-age=N` | Browser + CDN | Cache for N seconds |
| `s-maxage=N` | CDN only (browsers ignore) | CDN TTL — overrides max-age for shared caches |
| `public` | Browser + CDN | Explicitly cacheable by anyone including CDNs |
| `private` | CDN stays out | Only the browser can cache; CDN must not store |
| `no-cache` | Browser + CDN | CAN cache but MUST revalidate with origin before serving (via ETag/If-None-Match) |
| `no-store` | Browser + CDN | Never store anywhere — not browser, not CDN |
| `stale-while-revalidate=N` | Browser + CDN | After TTL expires, serve stale for N seconds while fetching fresh in background |
| `stale-if-error=N` | Browser + CDN | If origin is down, serve stale content for up to N seconds |

**WARNING — `no-cache` ≠ `no-store`:**
- `no-cache`: CDN stores a copy but checks with origin on every request. Origin replies `304 Not Modified` (no body, cheap) if unchanged.
- `no-store`: Never stored at all. Use for session tokens, financial data, PII.

---

## Key Patterns

### Different browser and CDN TTLs [CRITICAL]
```
Cache-Control: max-age=300, s-maxage=86400
```
- Browser caches for 5 minutes → deploy propagates to users quickly
- CDN caches for 24 hours → origin stays offloaded

### stale-while-revalidate (eliminates TTL latency spike) [IMPORTANT]
```
Cache-Control: max-age=60, stale-while-revalidate=30
```
- 0–60s: serve from cache instantly
- 60–90s: serve stale immediately, fetch fresh in background
- After 90s: must wait for origin

### stale-if-error (CDN-based resilience) [IMPORTANT]
```
Cache-Control: max-age=300, stale-if-error=3600
```
If origin is down, CDN serves stale content for up to 1 hour. Site stays up even during backend outages.

---

## Cache Key and the Vary Header [CRITICAL]

Default cache key: `Host + URL path + query string`

`Vary` tells the CDN to also vary the cache key by a request header:

| Header | Effect |
|---|---|
| `Vary: Accept-Encoding` | Separate cache entries for gzip vs Brotli. Safe and correct. |
| `Vary: Accept-Language` | Separate per language. Fine for a few languages; expensive for many. |
| `Vary: Cookie` | DANGER — creates one cache entry per unique cookie value. Session IDs are unique per user → one entry per user per URL → CDN caching completely defeated, storage explodes. |
| `Vary: User-Agent` | DISASTER — thousands of unique UA strings → cache explosion. Use separate URLs or responsive design instead. |

---

## Conditional Revalidation (ETag / 304) [IMPORTANT]

1. Origin sends: `ETag: "abc123"` with first response.
2. CDN caches the response + the ETag.
3. When TTL expires (or `no-cache`): CDN sends conditional request to origin:
   ```
   GET /logo.png
   If-None-Match: "abc123"
   ```
4. If unchanged: origin replies `304 Not Modified` — no body transferred. Cheap.
5. If changed: origin replies `200` with new body + new ETag.

`If-Modified-Since` (date-based) is the older alternative to `If-None-Match` (ETag-based). ETag is more precise.

---

## Cache Purging [CRITICAL]

TTL alone is insufficient for content that changes unpredictably. Use active purging.

### 1. URL purge
```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/purge_cache" \
  -H "Authorization: Bearer $CF_TOKEN" \
  --data '{"files": ["https://yoursite.com/products/42"]}'
```
Surgical — right for "one specific thing changed."

### 2. Tag-based purge (surrogate keys) [IMPORTANT]
Origin tags responses:
```nginx
add_header Cache-Tag "product-42, category-electronics, homepage-featured";
```
Purge by tag when data changes:
```bash
--data '{"tags": ["product-42"]}'
```
Invalidates every URL tagged `product-42` — product page, category listing, homepage widget — in one call. Cloudflare uses `Cache-Tag`. Fastly uses `Surrogate-Key`.

### 3. Wildcard purge
```bash
--data '{"prefixes": ["https://yoursite.com/products/"]}'
```
Dangerous — causes thundering herd. All those cache misses hit origin simultaneously.

---

## How to Detect Cache Behavior

| CDN | Cache HIT header | Cache MISS header |
|---|---|---|
| Cloudflare | `CF-Cache-Status: HIT` | `CF-Cache-Status: MISS` |
| Fastly | `X-Cache: HIT, HIT` | `X-Cache: MISS` |
| CloudFront | `X-Cache: Hit from cloudfront` | `X-Cache: Miss from cloudfront` |

---

## Nginx Config Example

```nginx
# Static assets — long CDN TTL, short browser TTL
location /static/ {
    add_header Cache-Control "public, max-age=300, s-maxage=86400, stale-while-revalidate=60, stale-if-error=3600";
    add_header Cache-Tag "static-assets";
}

# API responses — short TTL, ETag revalidation
location /api/products/ {
    add_header Cache-Control "public, max-age=60, s-maxage=300, stale-while-revalidate=30";
    etag on;
}

# User-specific — private, never CDN-cached
location /api/account/ {
    add_header Cache-Control "private, no-store";
}
```

---

## Trade-offs

| Scenario | Use CDN? | Why |
|---|---|---|
| Global users, static assets | Yes | 100–300ms latency reduction per asset |
| Global users, cacheable API | Yes | Origin offload + latency |
| User-specific API responses | Bypass caching (still proxy) | CDN can't cache; but TLS termination + DDoS protection still useful |
| Internal microservice API | No | CDN hop adds latency with no benefit |
| WebSockets | Depends | Cloudflare supports WS proxying; most CDNs don't |
| Sensitive/regulated data | No or `no-store` | Risk of CDN storing PII |

---

## Common Production Mistakes

1. **Using `max-age` without `s-maxage`:** Browser and CDN get the same TTL. Deploy a fix → users with cached browsers don't see it for hours. Fix: `max-age=300, s-maxage=86400`.

2. **No purge strategy for dynamic content:** Product prices, stock levels, seat availability cached with TTL. Changed at origin, still wrong at CDN for minutes/hours. Real financial and customer-trust risk.

3. **`Vary: Cookie`:** Turns your CDN into a per-user cache. Symptoms: CDN HIT rate near zero, origin traffic spikes, unexpected storage billing.

4. **Caching 301 redirects:** `301` is cacheable. If CDN caches a bad redirect at long TTL and you fix origin, CDN ignores the fix. Set `no-store` or short TTL on redirects that might change.

5. **Relying on CDN defaults for security:** If you forget `Cache-Control` on an authenticated endpoint, CDN default behavior varies by provider and page rule config. Always explicitly set `private, no-store` on user-specific or sensitive endpoints. Never rely on implicit behavior.

6. **One CDN, no failover:** Fastly 2021. Multi-CDN with health-check-based DNS failover is the resilience pattern for systems with availability SLOs.

---

## Quiz Questions and Answers

**Q1:** You set `Cache-Control: max-age=3600` on your homepage. You deploy a critical fix at 12:30. At 13:00 a new user visits — do they see the fix? What header change prevents this?

**A1:** They likely do NOT see the fix. The CDN cached the page at 12:00; at 13:00 the cache is still valid (TTL is 60 min, only 60 min have passed). Fix: use `max-age=300, s-maxage=3600` so browser refreshes every 5 min, AND add a deploy hook to purge the CDN cache on every deploy (URL purge or tag-based purge).

---

**Q2:** Your origin sends `Vary: Accept-Language` for a product page with 12 languages. Is this safe? What's the storage implication?

**A2:** Safe but expensive. CDN creates 12 separate cache entries per URL per language. At scale: 12x storage. Purging requires hitting all 12 variants — tag-based purge handles this cleanly (one purge by tag hits all variants), but URL-based purge misses them unless you enumerate all 12.

---

**Q3:** `/api/feed` returns personalized content. You deploy without a `Cache-Control` header. Cloudflare's default behavior + "Cache Everything" page rule is active. What happens?

**A3:** Cloudflare caches the first user's response under the cache key (URL). The next user hitting the same URL gets user A's personalized feed from cache — a **data leak**. Fix: always explicitly set `Cache-Control: private, no-store` on any user-specific or authenticated endpoint. Never rely on CDN default behavior as a security boundary.

---

## Follow-up Q&A

### Is an IP address the physical address of a machine?

No. There are two different address types:

| Address | Layer | What it identifies | Tied to hardware? |
|---|---|---|---|
| **MAC address** | Layer 2 (local network) | Physical network interface card | Yes — burned in at factory |
| **IP address** | Layer 3 (internet routing) | Logical routing label | No — assigned by configuration |

IP is a logical label. The routing table ties it to a destination. In Unicast, one IP is configured on one machine, so it *feels* physical — but that's just configuration, not a rule. Anycast proves it: the same IP configured on 50 machines in 50 cities. Nothing in the IP spec forbids this.

---

### How can two PoPs in different cities have the same IP? (Anycast internals)

Each PoP announces the same IP block via BGP simultaneously. Every ISP router on the internet receives multiple BGP announcements for the same prefix — one from each PoP. BGP picks the "best" path (shortest AS path, closest exit).

```
Jio Delhi router receives:
  104.21.0.0/16 via Cloudflare Delhi      → 1 hop  ← picked
  104.21.0.0/16 via Cloudflare Uttarakhand → 2 hops
  104.21.0.0/16 via Cloudflare Frankfurt   → 6 hops

BSNL Uttarakhand router receives:
  104.21.0.0/16 via Cloudflare Uttarakhand → 1 hop  ← picked
  104.21.0.0/16 via Cloudflare Delhi       → 2 hops
```

Same IP, two different physical machines served. The IP is a label; BGP routing decides which physical machine actually handles the packet.

**Anycast + TCP subtlety:** If BGP routing changes mid-connection, packets reroute to a different PoP that has no state for that TCP connection — connection resets. CDNs manage this risk because BGP changes are slow to propagate and most HTTP connections complete in milliseconds.

---

### How does the cache key get created? Does origin send it?

The origin does **not** send the cache key. The CDN computes it itself from the incoming request.

```
First request (MISS):
1. Request arrives at CDN edge
2. CDN computes key from request: "yoursite.com + /products/42"
3. Looks in local storage → nothing → MISS
4. Fetches response from origin
5. Stores response body under that key for the TTL duration
6. Serves to user

Every request after (HIT):
1. CDN computes same key: "yoursite.com + /products/42"
2. Found in local storage → HIT
3. Serves directly — origin never contacted
```

Responsibility split:
- CDN computes the cache key from the request
- Origin sets the TTL via `Cache-Control`
- CDN stores the response body under that key for that TTL

---

### Same URL, different users — does CDN treat them the same?

Yes. By default the cache key has no user identity — just Host + URL + query string. 1 million users hitting the same URL share one cached copy. Origin is contacted exactly once (on the first MISS).

```
User A → cache key: yoursite.com/products/42 → HIT (same copy)
User B → cache key: yoursite.com/products/42 → HIT (same copy)
...1 million users, 1 cached copy, origin contacted once
```

`Vary: Cookie` breaks this by appending the unique session cookie to the key. Every user gets a unique cache key → unique MISS → unique stored copy → 1 million entries for the same URL → CDN caching defeated.

---

### How do reads and writes work with a CDN? What happens when data changes?

**Writes always bypass the CDN cache and reach origin directly.**

CDNs only cache `GET` and `HEAD` by default. `POST`, `PUT`, `DELETE` are proxied straight to origin — CDN acts as a transparent pipe, stores nothing.

**The stale read problem:** After a write updates origin's data, the CDN still has the old cached GET response. Three fixes:

| Fix | How | When to use |
|---|---|---|
| TTL expiry | Do nothing — wait for TTL to run out | Acceptable for non-critical data |
| Active purge | App calls CDN purge API after every write | Critical data (prices, stock, seats) |
| `no-cache` + ETag | CDN revalidates with origin on every request via `If-None-Match` | Always-fresh with low origin cost |

```
Write flow:
User → CDN (pass-through) → Origin always

Read flow:
User → CDN → HIT (origin skipped)
           → MISS → Origin → CDN stores → User

After write: CDN cache is stale until purged or TTL expires
```

---

### Do you need Redis if you already have a CDN?

Yes — they operate at completely different layers and solve different problems.

```
User
  │
  ▼
[CDN edge]     ← stops request reaching origin at all
  │ (on MISS)
  ▼
[App server]
  │
  ▼
[Redis]        ← stops request reaching database
  │ (on MISS)
  ▼
[Database]
```

**What CDN cannot do that Redis handles:**

| Scenario | CDN helps? | Redis helps? |
|---|---|---|
| Public cacheable GET — 99% HIT rate | Yes | Yes (for the 1% MISS) |
| Authenticated routes (`/dashboard`, `/cart`) — CDN bypasses | No | Yes |
| Internal service-to-service calls | No | Yes |
| Session storage | No | Yes |
| Rate limiting counters | No | Yes |
| POST/PUT/DELETE requests | No (pass-through) | Yes (surrounding reads) |

Even with a 99% CDN HIT rate, the 1% that misses still hits your app server. Without Redis, all of those go straight to the database. With Redis, most are served from cache. And for all authenticated/private traffic that CDN never caches, Redis is the only cache layer protecting your database.
