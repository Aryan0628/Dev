# D26 — Redis Internals: Data Structures, Persistence (RDB vs AOF)

---

## Why Redis Exists

Your database lives on disk. Disk reads take ~1–10ms. Redis lives entirely in RAM. RAM reads take ~100 nanoseconds. That's 10,000x faster.

At scale, even one extra DB query per request becomes catastrophic. Redis is a fast scratch space in front of your database.

> **Real case: Twitter — 2012**
> What happened: Celebrity tweets caused millions of simultaneous timeline reloads. Each hit the DB for follower lists, tweets, user metadata — dozens of queries per request.
> Root cause: No caching layer. Every request went to the database cold.
> Impact: Database CPU at 100%, p99 latency in seconds, partial outages.
> What fixed it: Redis as a caching layer. DB load dropped ~80%.
> What it would have taken to prevent it: A caching strategy from day one, not retrofitted after things broke.

**p99 latency** = the latency that 99% of requests fall under. If p99 is 200ms, 1 in 100 requests takes longer. Averages hide slow outliers; p99 catches them.

---

## What Redis Is

Redis = **Re**mote **Di**ctionary **S**erver.

A giant dictionary (key → value) that lives entirely in memory, accessible over the network. Every operation executes in microseconds.

Analogy: Your database is a library with books in an archive (disk). Redis is a whiteboard on the wall. Whiteboard = 0.1 seconds. Archive = 10 minutes.

Redis is **not** a replacement for your database. It's a fast layer in front of it.

---

## Core Architecture [CRITICAL]

### Single-threaded command execution

Redis processes **one command at a time**. One thread runs all your GET, SET, LPUSH etc.

Why this isn't slow:
- Everything is in memory — operations finish in microseconds
- No disk I/O waiting
- Zero lock contention — no deadlocks, no race conditions on your data

Connects to D06: Redis uses the **event loop model** — I/O multiplexing via `epoll` (same as Nginx). I/O thread handles network; command thread executes commands.

Redis 6.0+ added I/O threads for reading/writing network data, but command **execution** is still single-threaded.

### Memory model — dict with progressive rehashing

All keys live in a **hash table** (`dict`). A key like `"user:123:name"` hashes to a bucket, value stored there.

When the hash table load factor exceeds 1, Redis performs **progressive rehashing**:
- Creates a second, larger hash table
- Gradually moves entries from old to new over multiple operations
- Avoids a long freeze during resize

```
dict[0] → [bucket 0] ... [bucket N]       ← old table, being drained
dict[1] → [bucket 0] ... [bucket 2N]      ← new table, being filled
```

During rehashing: reads check both tables. Writes go to new table only.

---

## Data Structures [IMPORTANT]

The key is always a string. The **value** type determines internal representation.

### 1. String [CRITICAL]

```
SET user:123:name "Aryan"
GET user:123:name       → "Aryan"
INCR page:views         → 1  (atomic — no race condition)
```

Internally: small strings use embedded string (stored directly in object, no heap pointer). Larger strings use **SDS** (Simple Dynamic String) — stores length explicitly so `strlen()` is O(1).

Use for: counters, session tokens, JSON blobs, feature flags.

**Common mistake**: storing `SET user:123 '{"name":"Aryan","age":20}'`. To update one field, you GET the whole blob, decode, modify, encode, SET back. Use a Hash instead.

### 2. Hash

```
HSET user:123 name "Aryan" age 20 city "Delhi"
HGET user:123 name          → "Aryan"
HGETALL user:123            → all fields
HINCRBY user:123 age 1      → 21
```

Internally: **listpack** (compact byte array) for small hashes (< 128 fields, values < 64 bytes). Switches to real hash table when larger.

Use for: user objects, config objects — anything with multiple named fields.

### 3. List

```
RPUSH notifications:123 "new follower" "new like"
LPOP notifications:123     → "new follower"
LRANGE notifications:123 0 -1  → all items
```

Internally: listpack for small lists, **quicklist** (linked list of listpacks) for large lists.

Use for: queues (RPUSH + LPOP), activity feeds, recent history.

### 4. Set

```
SADD online:users "user:1" "user:2"
SISMEMBER online:users "user:1"   → 1 (yes)
SCARD online:users                → 2 (count)
SINTER online:users active:users  → intersection
```

Internally: **intset** (compact array) for integer-only sets, hash table otherwise.

Use for: unique visitor tracking, friend lists, tag systems, deduplication.

### 5. Sorted Set (ZSet) [IMPORTANT]

```
ZADD leaderboard 9500 "aryan" 8200 "priya"
ZRANK leaderboard "aryan"           → 0 (rank 0 = highest)
ZRANGE leaderboard 0 2 WITHSCORES  → top 3 with scores
ZINCRBY leaderboard 100 "priya"    → 8300
```

Internally: **skiplist + hash table**. Skiplist keeps entries sorted by score for range queries. Hash table lets you look up a member's score in O(1).

Skiplist = layered linked list. Start high, skip large chunks to find target:
```
Layer 2:  [head] ────────────────→ [5000] → [9500]
Layer 1:  [head] → [2000] → [5000] → [8200] → [9500]
Layer 0:  [head] → [1100] → [2000] → [3300] → [5000] → [8200] → [9100] → [9500]
```

Use for: leaderboards, rate limiting (sliding window), job priority queues.

### Summary Table

| Type | Use case | Internal structure |
|---|---|---|
| String | Counter, token, blob | SDS / embedded string |
| Hash | User object, config | Listpack → Hash table |
| List | Queue, activity feed | Quicklist (listpack chunks) |
| Set | Unique tracking, tags | Intset → Hash table |
| Sorted Set | Leaderboard, rate limit | Skiplist + Hash table |

---

## Persistence [CRITICAL]

Redis is in-memory. When the server restarts, RAM is wiped. Persistence saves data to disk so Redis can reload on startup.

---

### RDB — Point-in-time Snapshots

Creates a binary `.rdb` snapshot of the entire dataset.

**How it works:**
```
Redis parent process
      |
      | fork()  ← OS call — creates a copy of the process
      |
 ┌────┴────┐
 │  Child  │ → writes entire dataset to dump.rdb
 │ process │ → exits when done
 └─────────┘
      |
 Parent keeps serving requests while child writes (copy-on-write)
```

`fork()` uses **copy-on-write**: child sees data as it was at fork time. If parent modifies a memory page, OS copies that page first — child still sees the original. No freeze.

```
save 900 1       # save if ≥1 key changed in 900 seconds
save 300 10      # save if ≥10 keys changed in 300 seconds
save 60 10000    # save if ≥10000 keys changed in 60 seconds
```

Pros: small file, fast restart, minimal I/O impact.
Cons: can lose data between snapshots (up to minutes).

---

### AOF — Append Only File

Logs every write command to a file. On restart, replays the log.

```
SET user:1 "aryan"   → appended to appendonly.aof
INCR page:views      → appended to appendonly.aof
```

Three fsync options (fsync = OS call that forces kernel to flush write buffer to actual disk):

| `appendfsync` | Meaning | Data loss risk | Performance |
|---|---|---|---|
| `always` | fsync after every command | None (1 command max) | Slow |
| `everysec` | fsync every 1 second | Up to 1 second | Fast (default) |
| `no` | Let OS decide (~30s) | Up to 30 seconds | Fastest |

Without fsync, data is in the OS page cache — survives Redis crash but not OS crash.

**AOF Rewrite** — the log grows forever. Redis periodically compacts it automatically:
```
SET x 1
SET x 2
SET x 3
→ rewrite produces just: SET x 3
```

AOF rewrite is **not a separate toggle** — it's built into AOF mode. It triggers automatically based on config:
```
auto-aof-rewrite-percentage 100   # rewrite when AOF is 100% bigger than last rewrite
auto-aof-rewrite-min-size 64mb    # only if AOF is at least 64MB
```

Set `auto-aof-rewrite-percentage 0` to disable auto-rewrite. Manual trigger: `BGREWRITEAOF`.

AOF rewrite also uses `fork()` — same copy-on-write trick as RDB. AOF rewrite has nothing to do with RDB in normal mode.

Pros: minimal data loss (1 second worst case), human-readable log.
Cons: larger file than RDB, slower restart.

---

### Hybrid Persistence (RDB + AOF) — Recommended for Production [IMPORTANT]

```
aof-use-rdb-preamble yes
```

In hybrid mode, AOF rewrite changes its output format:
- Normal AOF rewrite: pure command log
- Hybrid AOF rewrite: **RDB snapshot at the front** + AOF commands for the delta

On restart:
1. Redis opens `.aof` file
2. Reads RDB preamble (fast bulk load)
3. Replays AOF commands at the tail (just the delta since last rewrite)

The `.rdb` file and `.aof` file are still **two separate files on disk**. Hybrid mode changes the internal format of the `.aof` file — it embeds RDB at its start.

---

### Where Files Are Saved

Both files live on **disk**, in the directory Redis is configured to use:

```
# redis.conf
dir /var/lib/redis           ← directory for both files
dbfilename dump.rdb          → /var/lib/redis/dump.rdb
appendfilename appendonly.aof → /var/lib/redis/appendonly.aof
```

```
RAM (volatile — lost on restart)
┌─────────────────────────────────┐
│  Redis in-memory dataset        │
│  (dict hash table)              │
└─────────────────────────────────┘
          ↓ persistence writes
Disk (durable — survives restart)
┌─────────────────────────────────┐
│  /var/lib/redis/                │
│  ├── dump.rdb       ← snapshot  │
│  └── appendonly.aof ← cmd log   │
└─────────────────────────────────┘
```

In production: point `dir` at a separate attached disk (EBS on AWS, Persistent Disk on GCP) — not the ephemeral OS disk. An OS reinstall or instance failure wipes ephemeral storage.

---

### No Persistence

Pure cache mode — Redis starts empty on restart, re-warms from DB.

```
save ""         ← disable RDB
appendonly no   ← disable AOF
```

---

### Persistence Comparison Table

| Mode | Data loss | Restart speed | File size | Use when |
|---|---|---|---|---|
| No persistence | All data | Instant | 0 | Pure cache |
| RDB only | Up to minutes | Fast | Small | Cache, tolerable loss |
| AOF (everysec) | Up to 1 second | Slower | Larger | Near-durable |
| AOF (always) | Zero | Slowest | Largest | Must not lose any write |
| Hybrid | Up to 1 second | Fast | Medium | Production default |

---

## Trade-offs and When NOT to Use Redis [IMPORTANT]

**When NOT to use Redis:**
- Data must survive crash with zero loss → use PostgreSQL with WAL (`fsync=on`)
- Dataset larger than RAM → memory is expensive; 100GB+ becomes cost-prohibitive
- Complex queries needed → no JOIN, no GROUP BY, no WHERE across multiple keys
- Strong consistency across multiple keys → `MULTI/EXEC` transactions exist but no rollback on logic errors

**Most common production mistakes:**

1. **No maxmemory set** — Redis fills RAM, OS starts swapping to disk, Redis becomes slower than the database you were trying to avoid. Always set `maxmemory` + eviction policy.

2. **Using Redis as primary storage but treating persistence as optional** — if Redis is the source of truth, run it in AOF/hybrid mode. If it's a cache, DB is the source of truth and pure cache mode is fine. Decide which one it is — never leave it ambiguous.

---

## Eviction Policies [GOOD TO KNOW]

When Redis hits `maxmemory`, it evicts keys to make room:

| Policy | What gets evicted |
|---|---|
| `noeviction` | Reject writes (error) |
| `allkeys-lru` | Least Recently Used across all keys |
| `volatile-lru` | LRU but only among keys with TTL set |
| `allkeys-random` | Random key |
| `volatile-ttl` | Key expiring soonest |

**LRU** (Least Recently Used) = the key not accessed for the longest time gets evicted first.

For a cache in front of a database: use `allkeys-lru`. Redis automatically drops cold data to make room for hot data.

---

## Quiz Questions + Answers

**Q1.** Redis is single-threaded for command execution. You have a Sorted Set with 10 million members. You run `ZRANGE leaderboard 0 9` (top 10). Does the single-threaded model cause any problem here?

**A1.** No problem for this specific query. ZRANGE on a skiplist is O(log N + M) where N = total members, M = members returned. For top 10 out of 10M, that's roughly 23 steps on the skiplist + 10 returns — microseconds. The single-threaded model only becomes a problem when a single command takes a long time (e.g., `KEYS *` on millions of keys blocks all other requests). ZRANGE of a small range does not.

---

**Q2.** You have two options for a session store (e-commerce, logged-in cart):
- Option A: RDB with saves every 5 minutes
- Option B: AOF with `appendfsync everysec`

Which do you pick and why? What's the exact worst case data loss?

**A2.** Option B. Sessions are worth money — a lost cart means lost revenue. Worst case data loss: Option A = up to 5 minutes of sessions (everyone who logged in in the last 5 minutes gets logged out). Option B = up to 1 second of writes. For a session store, 1 second is acceptable; 5 minutes is not. Use AOF with `everysec` + `aof-use-rdb-preamble yes` for fast restarts.

---

**Q3.** A developer says: "We use Redis as our primary database because it's 10,000x faster than PostgreSQL." What two specific risks do you raise, and what config changes do you require?

**A3.**
- Risk 1: **Data loss on crash** — if running with no persistence or RDB-only, a crash loses data. Redis as primary DB requires AOF with `appendfsync always` (zero loss) or at minimum `everysec` (1 second loss). Require: `appendonly yes` + `appendfsync everysec` minimum, `aof-use-rdb-preamble yes`.
- Risk 2: **Memory capacity** — Redis stores everything in RAM. As data grows, you either hit `maxmemory` (evictions start deleting data you thought was permanent) or RAM costs become extreme. If `maxmemory` is not set, Redis will consume all available RAM and the OS will start swapping, making it slower than PostgreSQL. Require: `maxmemory` set explicitly, and `maxmemory-policy noeviction` (so writes fail loudly rather than silently deleting data).
