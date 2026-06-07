# D32 — Caching Strategies: Cache-Aside, Write-Through, Write-Behind

---

## Why It Matters

Redis is 10,000x faster than disk. But knowing Redis exists and knowing **how to use it correctly** are different things. The wrong strategy causes either stale data (users see wrong prices, old inventory) or data loss (writes that never reach the database).

> **Real case: Facebook Memcached — 2013**
> What happened: When a popular cache key expired or a server restarted, millions of simultaneous requests found an empty cache and all slammed the database at once.
> Root cause: Cache-aside with no coordination on misses. Every server independently decided "cache is empty, I'll fetch from DB."
> Impact: Database CPU spiked to 100% within milliseconds. Partial outages on profile pages.
> What fixed it: Leases — a token system where only one server gets permission to fetch from DB on a miss; others wait. (Covered in D33 — cache stampede.)

---

## Three Core Terms

- **Cache hit** — data is in cache. Fast path. ~100µs.
- **Cache miss** — data not in cache. Falls back to DB. ~10ms.
- **TTL** (Time To Live) — how long a key lives before Redis auto-deletes it.

```
SET user:123 "aryan" EX 300    ← TTL of 300 seconds (5 minutes)
```

After 300s, Redis deletes the key. Next `GET user:123` → nil (miss).

---

## Strategy 1 — Cache-Aside (Lazy Loading) [CRITICAL]

Most common pattern. The **app** owns all cache logic. Cache is a dumb fast store.

### Read path:
```
App                  Cache (Redis)         Database
 |── GET user:123 ──>|                     |
 |   [HIT] ←─────── | returns value        |
 |   return          |                     |
 |                   |                     |
 |   [MISS] ────────nil                    |
 |────────────────────────── SELECT * ────>|
 |<────────────────────────── row data ────|
 |── SET user:123 EX 300 ─>|               |
 |   return to user         |               |
```

```python
def get_user(user_id):
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)         # cache hit

    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.set(f"user:{user_id}", json.dumps(user), ex=300)
    return user                           # cache miss — now cached
```

### Write path:
```python
def update_user(user_id, new_name):
    db.query("UPDATE users SET name = ? WHERE id = ?", new_name, user_id)
    redis.delete(f"user:{user_id}")       # invalidate — don't update, delete
```

Delete on write, not update. Simpler and safer — next read re-populates from DB.

### Pros:
- Only caches what's actually read — no wasted memory
- Cache failure doesn't break the app — falls back to DB
- Simple to implement

### Cons:
- First request is always slow (cold cache)
- Stale data window between DB write and cache invalidation
- Cache stampede on popular key expiry (→ D33)

**[CRITICAL] Most common mistake:** No TTL set. Keys live forever → stale data never clears → memory grows unboundedly → Redis hits maxmemory → unpredictable evictions.

**Always set a TTL.**

---

## Strategy 2 — Write-Through [IMPORTANT]

Every write goes to **both** cache and DB **synchronously**. Both succeed before returning to caller.

```
App                  Cache (Redis)         Database
 |── write data ────>|                     |
 |── SET key ───────>| ✓                   |
 |────────────────────────── UPDATE ───────>|
 |<────────────────────────── ✓ ───────────|
 |   write complete   |                     |
```

```python
def update_user(user_id, new_name):
    db.query("UPDATE users SET name = ? WHERE id = ?", new_name, user_id)
    redis.set(f"user:{user_id}", json.dumps({"name": new_name}), ex=86400)
    # both done synchronously
```

### Pros:
- Cache always consistent with DB — no stale data
- Fresh immediately after every write

### Cons:
- Slightly higher write latency (two writes per operation)
- Cache fills with data that may never be read (cold writes waste memory)

**When to use:** Read-heavy, stale data unacceptable. Product pages, user profiles.
**When NOT to use:** Write-heavy systems; unpredictable write patterns that fill cache with cold data.

---

## Strategy 3 — Write-Behind (Write-Back) [IMPORTANT]

Write goes to **cache only**, returns success immediately. DB write happens async in background.

```
App                  Cache (Redis)         Database
 |── write data ────>|                     |
 |── SET key ───────>| ✓                   |
 |   return success   |                     |
 |   (DB not yet)     |                     |
 |              [background worker]         |
 |                    |── UPDATE ──────────>|
 |                    |<── ✓ ──────────────|
```

### Pros:
- Extremely fast writes — only waits for Redis (~0.1ms)
- Reduces DB write pressure — can batch writes

### Cons:
- **Data loss risk** — if Redis crashes before worker flushes, writes are gone
- Complex background worker needed (retry logic, ordering, crash recovery)
- Other servers reading from DB directly see stale data

**When to use:** High-volume writes where small data loss is acceptable. View counters, like counts, analytics.
**When NOT to use:** Anything involving money, inventory, user account data.

**On RDB/AOF + write-behind:** AOF does persist Redis state — if Redis recovers from AOF, the key comes back. But AOF everysec still loses up to 1 second of writes, AND your worker still needs to identify what's in Redis but not in DB and replay it correctly. This is complex and risky. For financial data, use write-through instead — don't try to make write-behind safe with AOF.

---

## Strategy 4 — Read-Through [GOOD TO KNOW]

Same as cache-aside but the **cache layer itself** handles the miss — not the app. App only ever talks to cache, never knows about hits/misses.

Less common in practice. Redis doesn't support this natively — needs a caching library or proxy layer. Cache-aside is more explicit and easier to debug.

---

## Comparison Table

| Strategy | Who loads cache | DB write | Stale data risk | Data loss risk | Write speed |
|---|---|---|---|---|---|
| Cache-aside | App (on miss) | App (direct, sync) | Yes (TTL window) | None | Fast |
| Write-through | App (on write) | App (sync) | No | None | Slightly slower |
| Write-behind | App (on write) | Worker (async) | Yes | Yes (crash) | Very fast |
| Read-through | Cache layer | App (direct) | Yes (TTL window) | None | Fast |

---

## Redis Data Structure → Use Case Mapping (from D26)

| What you cache | Redis type | Why |
|---|---|---|
| User profile, product detail | String (JSON) | Simple GET/SET |
| User object, many fields | Hash | Update one field without GET+SET whole blob |
| Recent activity feed | List | RPUSH new, LTRIM to keep last N |
| Online users | Set | SADD/SREM/SISMEMBER |
| Leaderboard, rate limit | Sorted Set | ZADD with score, ZRANGE for top N |

---

## Real Production Pattern

```
User profile read       → cache-aside
User profile update     → write-through  (can't show stale name/photo)
Page view counter       → write-behind   (losing 1 second of views is fine)
Shopping cart           → cache-aside + AOF persistence on Redis
```

---

## Quiz Questions + Answers (with Aryan's answers)

**Q1.** Product page (read-heavy, updated once a day by admins). Which strategy, what TTL, what happens on write?

**Aryan's answer:** Write-through — correct. Less writes, more reads, no stale data acceptable. Higher TTL reduces cold misses since product changes rarely.

**Full answer:** Write-through is correct. For TTL: 12–24 hours is reasonable — product rarely changes, so long TTL means entries survive between admin updates without expiring and hitting the DB unnecessarily. On write path: admin updates price → `db.query(UPDATE ...)` + `redis.set(f"product:{id}", updated_json, ex=86400)` — both synchronous before returning success.

---

**Q2.** Developer uses write-behind for user account balances (fintech). Risk?

**Aryan's answer:** Redis crash before worker flushes → DB never gets write → money lost. Mentioned RDB/AOF could help.

**Full answer:** Correct danger. On RDB/AOF: AOF can recover the Redis key after crash, but (a) AOF everysec still loses up to 1 second, (b) your worker must correctly replay what's in Redis-but-not-in-DB after recovery — that logic is complex and failure-prone. For financial data, **never use write-behind**. Use write-through: both Redis and DB written synchronously before returning success. If either fails, the whole write fails — no silent loss.

---

**Q3.** Cache-aside with no TTL set. Three problems after 3 months?

**Aryan's answer:** Keys don't expire (correct — stale data). Missed two.

**Full answer:**
1. **Stale data** — DB updated but cache key has old value forever. Users see wrong prices, old names, stale inventory.
2. **Memory grows unboundedly** — every key ever written accumulates. Redis hits `maxmemory`. If `noeviction` → writes error. If `allkeys-lru` → Redis silently deletes data with no warning to the app.
3. **Cold data crowds out hot data** — keys written once 3 months ago still sit in cache, never accessed. Hot frequently-read keys compete for memory with dead cold keys. Cache efficiency drops.

**Fixes:**
- Set TTL on every key (clears stale data + cold data naturally)
- Set `maxmemory` + `maxmemory-policy allkeys-lru` (caps memory, keeps hot data)
