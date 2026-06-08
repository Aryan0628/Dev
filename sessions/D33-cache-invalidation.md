# D33 — Cache Invalidation: TTL, Event-Driven, Cache Stampede & Thundering Herd

---

## The Hard Problem

> *"There are only two hard things in Computer Science: cache invalidation and naming things."*
> — Phil Karlton, Netscape

Your cache is a **copy** of real data. The moment the original changes, your copy is wrong. Cache invalidation is how you know when to throw the copy away.

Get it wrong → users see stale prices, outdated inventory, wrong balances.
Over-invalidate → cache hit rate drops to zero, Redis complexity added for nothing.

> **Real case: Amazon — pricing bug**
> What happened: Product pages showed prices hours out of date. Users added items to cart at the displayed price, saw a different price at checkout.
> Root cause: Cache TTLs set too long. Price updates in DB took hours to propagate to cached product pages.
> Impact: Abandoned carts, customer complaints, eroded trust.
> What it would have taken to prevent it: Event-driven invalidation — delete the cache key the moment price updates in DB. Not rely on TTL expiry alone.

---

## Method 1 — TTL-Based Invalidation [CRITICAL]

A key expires after N seconds. Next request gets a miss, fetches fresh data from DB, re-populates cache.

```
SET product:123 "{price: 999}" EX 300    ← expires in 5 minutes
```

**Passive** invalidation — you're not telling the cache "this is stale." You're saying "assume stale after X seconds."

### How Redis implements TTL internally

Redis does NOT scan all keys every second. That would be O(N) and block everything. Two mechanisms:

**1. Lazy expiry** — when you `GET` a key, Redis checks if it's expired. If yes, deletes it and returns nil. Key is not deleted until accessed.

**2. Active expiry** — every 100ms, a background job randomly samples 20 keys with TTLs. Deletes expired ones. If >25% of sampled keys were expired, repeats immediately.

```
Every 100ms:
  sample 20 random keys with TTL
  delete expired ones
  if >25% expired → repeat immediately
  else → wait for next 100ms cycle
```

A key may live **slightly past its TTL** if unaccessed and unsampled. In practice: milliseconds to seconds overshoot. Usually fine.

### Choosing TTL

| Data type | Change frequency | Recommended TTL |
|---|---|---|
| Product price | Multiple times/day | 5–15 minutes |
| User profile | Rarely | 1–24 hours |
| Session token | Per session | Session length |
| Sports score / stock | Every second | 1–5 seconds |
| Static config | Almost never | 24 hours+ |

**The trap:** Setting TTL too long to "reduce DB load." If price changes hourly but TTL is 24 hours, users see stale prices for up to 24 hours. TTL is a safety net, not your primary strategy for frequently-changing data.

---

## Method 2 — Event-Driven Invalidation [CRITICAL]

**Actively delete the cache key the moment data changes in the DB.** No waiting for TTL.

```python
def update_product_price(product_id, new_price):
    db.query("UPDATE products SET price = ? WHERE id = ?", new_price, product_id)
    redis.delete(f"product:{product_id}")    # immediate invalidation
```

Next read gets a miss → fetches fresh data → re-populates. No stale window.

### Is this the same as cache-aside?

**Read path: identical to cache-aside.** Miss → fetch from DB → populate cache.

The difference is only in the **write path**, and it only matters in microservices:

**Cache-aside (single service):**
```
App
 |── UPDATE DB ──> DB
 |── redis.delete ──> Redis
Same service does both. Synchronous. Simple.
```

**Event-driven (microservices — different services own the write vs the cache):**
```
Price Service       Queue         Product Service (owns cache)
     |── UPDATE ──> DB              |
     |── publish ──> queue ────────>|── redis.delete ──> Redis
```

The writer doesn't own the cache → can't call `redis.delete()` directly → publishes an event → cache owner listens and invalidates.

**One-line answer:** Event-driven invalidation is cache-aside's write path adapted for microservices. Same read path. Different who does the invalidation and how.

### TTL still needed even with event-driven

Events can fail — queue down, consumer crashes, network partition. TTL is your fallback.

```python
redis.set(f"product:{product_id}", json.dumps(product), ex=3600)  # 1 hour safety net
# + event-driven delete on every update
```

Normal path: instant invalidation via event.
Worst case: TTL cleans up within 1 hour.

---

## Does Write-Through Need Cache Invalidation?

**Mostly no** — and this is the key advantage of write-through.

With write-through, every update writes to both cache and DB synchronously. Cache is always fresh. So you **don't need event-driven invalidation or active deletion on updates**.

But two things still apply:

**1. TTL — still needed, but for a different reason:**
- Cache-aside TTL = correctness (clears stale data)
- Write-through TTL = **memory management only** (clears cold/old data that accumulates)
- Without TTL, products nobody reads in 6 months sit in cache forever → Redis hits maxmemory

**2. Deletes still need explicit handling:**
```python
def delete_product(product_id):
    db.query("DELETE FROM products WHERE id = ?", product_id)
    redis.delete(f"product:{product_id}")    # still needed
```

Write-through covers updates. Deletes are still a manual invalidation step.

| Scenario | Cache-aside | Write-through |
|---|---|---|
| Update in DB | Need to delete cache key | Handled automatically |
| Delete from DB | Need to delete cache key | Still need to delete cache key |
| TTL purpose | Correctness + memory | Memory only |
| Stampede risk | Yes (on TTL expiry) | Lower (cache stays fresh longer) |

---

## Method 3 — Cache Stampede (Thundering Herd) [CRITICAL]

The most common cache-related outage pattern in production.

### What happens

```
Popular key "product:iphone" expires at 12:00:00.000

12:00:00.001 — Request A: GET product:iphone → nil (miss)
12:00:00.002 — Request B: GET product:iphone → nil (miss)
12:00:00.003 — Request C: GET product:iphone → nil (miss)
... 1000 more requests all get nil

All 1000 simultaneously:
  → query DB for product:iphone
  → DB gets 1000 identical queries at once
  → DB CPU spikes to 100%
  → DB slows, all queries take longer
  → unrelated queries start timing out
  → cascading failure
```

The cache was supposed to protect the DB. Instead, on expiry, it sent a 1000x burst directly to the DB.

**Thundering herd** = herd of cattle (requests) in a pen (cache). Gate opens (miss) → all stampede through the same narrow passage (DB) at once.

---

### Fix 1 — Mutex / Lock [CRITICAL]

Only **one** request fetches from DB. The rest wait.

```python
def get_product(product_id):
    key = f"product:{product_id}"
    lock_key = f"lock:{key}"

    cached = redis.get(key)
    if cached:
        return json.loads(cached)

    # NX = "set only if Not eXists" — atomic, only one request wins
    acquired = redis.set(lock_key, "1", nx=True, ex=5)

    if acquired:
        # I won — fetch from DB
        product = db.query("SELECT * FROM products WHERE id = ?", product_id)
        redis.set(key, json.dumps(product), ex=300)
        redis.delete(lock_key)
        return product
    else:
        # Someone else is fetching — wait and retry
        time.sleep(0.05)
        return get_product(product_id)   # by now cache is populated
```

`NX` (Not eXists) is atomic in Redis — only one caller wins even under simultaneous requests.

**Watch out:** If the lock `EX` is 5 seconds but the DB query takes 8 seconds, the lock expires before the DB query finishes. A second request acquires the lock, fires another DB query. Now two requests hit DB simultaneously. Fix: set lock TTL generously above your worst-case DB query time.

---

### Fix 2 — Probabilistic Early Expiry (XFetch) [IMPORTANT]

Start refreshing the key **before** it expires — probabilistically. Closer to expiry = higher refresh chance.

```
Key TTL: 300 seconds
At 295s remaining → ~1% chance of early refresh
At 10s remaining  → ~50% chance
At 1s remaining   → ~99% chance
```

One request rolls the dice and wins early refresh. All others still get a cache hit (key not expired yet). Winner refreshes in background. Key never expires under load — no stampede ever happens.

---

### Fix 3 — Stale-While-Revalidate [IMPORTANT]

Serve the **stale value immediately** while refreshing cache in background.

```python
def get_product(product_id):
    key = f"product:{product_id}"
    stale_key = f"stale:{key}"

    cached = redis.get(key)
    if cached:
        return json.loads(cached)        # normal hit

    stale = redis.get(stale_key)         # real key expired — check stale copy
    if stale:
        # Serve stale immediately, refresh async
        threading.Thread(target=refresh_cache, args=(product_id,)).start()
        return json.loads(stale)         # slightly old but instant response

    return fetch_from_db_and_cache(product_id)  # cold start, must wait
```

Two keys per entry: real key (short TTL) + stale key (long TTL). When real expires, serve stale while refreshing. Users get ~100µs response instead of ~10ms DB wait.

---

## Thundering Herd Beyond Caches

Same pattern appears everywhere — N things wait for one event, all wake simultaneously:

| Scenario | Fix |
|---|---|
| 1000 servers retry failed API at same moment | Exponential backoff + random jitter |
| 1000 servers run same cron job at same second | Randomised startup delay |
| All DB connections timeout and reconnect simultaneously | Staggered reconnect with jitter |

The fix is always the same: **desynchronise** the burst.

---

## Full Comparison Table

| Method | Stale window | Complexity | DB protection on expiry |
|---|---|---|---|
| TTL only | Up to TTL duration | Low | None |
| Event-driven | Near zero | Medium | None |
| TTL + event-driven | Fallback only | Medium | None |
| Mutex on miss | Near zero | Medium | Yes — serialises misses |
| Stale-while-revalidate | One request cycle | Medium | Yes — never a cold miss |
| Probabilistic early expiry | Zero | High | Yes — key never expires under load |

**Best production combo:**
```
Event-driven invalidation     ← primary: instant on every write
+ TTL as safety net           ← fallback: catches missed events
+ Mutex or stale-while-revalidate ← protects DB on high-traffic key expiry
```

---

## Quiz Questions + Model Answers

**Q1.** Product page cached with 5-minute TTL. Flash sale — 50,000 users hit the page simultaneously. TTL expires during peak traffic. What happens to your DB, and which fix do you apply?

**A1.** All 50,000 requests get a cache miss simultaneously. All fire the same DB query at once. DB CPU spikes to 100%. Unrelated queries start timing out. Potential cascading failure across the whole system — not just product pages.

Fix: **Mutex**. Set a Redis lock (`NX + EX`). Only one of the 50,000 requests wins the lock and hits the DB. The other 49,999 wait ~50ms and retry — by then the winner has populated the cache. Everyone gets a response, DB gets one query instead of 50,000.

Alternatively: **stale-while-revalidate** — if the stale key exists, all 50,000 get served the stale value instantly while one background thread refreshes. Zero DB burst.

---

**Q2.** Teammate: "Skip event-driven — just use 10 second TTL. That's close enough to real-time." Problem?

**A2.** Correctness problem: 10 second staleness may still be unacceptable (price changes, inventory going to zero). But the bigger production problem is **stampede frequency**. With 5-minute TTL, a popular key stampedes once every 5 minutes. With 10-second TTL, the same key stampedes every 10 seconds — 30x more often. At 10,000 requests/second on that key, that's 10,000 simultaneous DB hits every 10 seconds. Short TTL + high traffic = constant stampede. You've made the system less stable, not more consistent.

---

**Q3.** Mutex with `EX 5`. DB query takes 8 seconds. What goes wrong?

**A3.** Lock expires after 5 seconds. DB query is still running (seconds 5–8). A second request finds no lock (`NX` succeeds) and acquires it. Second request fires another identical DB query. Now two DB queries run simultaneously for the same key. When the first finishes (at 8s) it writes to cache and deletes the lock. When the second finishes (at ~13s) it overwrites cache with the same data and tries to delete the lock (already gone — no-op). In this specific case the result is correct but you got two DB hits instead of one — stampede protection partially failed.

Worse scenario: if 5 more requests each acquire the lock during the 8-second window before the first finishes, you get 6 simultaneous DB queries. The fix: set lock `EX` to at least 2x your worst-case DB query time. If queries can take up to 5s, set `EX 10` or `EX 15`.
