# D15 — SQL Fundamentals for Production

**Topics:** Indexes, EXPLAIN/EXPLAIN ANALYZE, N+1 Problem  
**Priority labels:** All three sections are [CRITICAL]

---

## The Anchor Incident

> **Real case: Friendster — 2003–2004**
> What happened: Friendster was the first major social network. Pages took 30–40 seconds to load. Users left for MySpace.
> Root cause: Database queries doing full table scans on 100M+ row tables with no indexes on join columns. N+1 query patterns in the friend-feed logic made it worse.
> Cost / impact: Friendster never recovered. Sold for $26M. Engineers estimated the DB problems were fixable in a week.
> Prevention: Composite indexes on the relationships table + fixing the N+1 pattern.

---

## Part 1 — Indexes [CRITICAL]

### The Analogy
A SQL index is a book's back-of-page index. Without it: read every row. With it: jump directly to matching rows.

### B-Tree Structure
B-Tree = Balanced Tree. Every leaf node is the same depth from root → O(log n) lookups guaranteed.

```
                       [50]
                      /    \
              [25]              [75]
             /    \             /    \
        [10,20]  [30,40]   [60,70]  [80,90,95]
           ↓         ↓         ↓         ↓
        heap       heap      heap      heap
        pages      pages     pages     pages
```

- **Internal nodes**: key values + pointers to children
- **Leaf nodes**: key values + `ctid` (physical location: page_number + slot_number)
- **Heap pages**: actual table data, stored separately

Lookup of `WHERE id = 60`: root → right subtree → left child → leaf → ctid → heap page. 3 comparisons (23 for 10M rows in reality).

### What the Index Stores
```
Index leaf page:
  ['alice@ex.com'  | ctid: (page 4,  slot 2)]
  ['bob@ex.com'    | ctid: (page 7,  slot 1)]
  ['john@ex.com'   | ctid: (page 11, slot 3)]
```
Sorted alphabetically for string columns. Sorted numerically for number columns.

**One-line answer:** An index is a sorted B-Tree of one or more columns with pointers to the real rows, enabling O(log n) lookups instead of O(n) full scans.

### Creating Indexes Safely
```sql
-- NEVER on a large live table (acquires full lock):
CREATE INDEX idx_users_email ON users(email);

-- Always use CONCURRENTLY in production:
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
-- Slower to build but doesn't block reads or writes
```

### When PostgreSQL Ignores Your Index
1. **Table is small** — Seq Scan is faster under ~1,000 rows
2. **Fetching >10–15% of rows** — sequential I/O beats random I/O for large result sets
3. **Function on the indexed column:**
   ```sql
   -- Index on email is NOT used:
   WHERE LOWER(email) = 'john@example.com'
   
   -- Fix: functional index
   CREATE INDEX idx_users_email_lower ON users(LOWER(email));
   -- Now the index IS used
   ```
   Same trap: `WHERE created_at::date = '2024-01-01'` — the `::date` cast breaks index usage
4. **Stale statistics** — run `ANALYZE users;` to refresh

### Most Common Mistakes
1. Using `WHERE LOWER(col) = ...` and wondering why the index doesn't work
2. Running `CREATE INDEX` (without CONCURRENTLY) on a large live table — locks it for minutes

---

## Part 2 — EXPLAIN and Query Plans [CRITICAL]

### EXPLAIN vs EXPLAIN ANALYZE

| Command | What it does | Runs query? |
|---|---|---|
| `EXPLAIN` | Shows the plan PostgreSQL intends to use | No |
| `EXPLAIN ANALYZE` | Plan + actual execution stats | Yes |
| `EXPLAIN (ANALYZE, BUFFERS)` | Also shows cache hit/miss | Yes |

**Warning:** `EXPLAIN ANALYZE DELETE ...` actually deletes rows. Wrap in a transaction:
```sql
BEGIN;
EXPLAIN ANALYZE DELETE FROM users WHERE id = 1;
ROLLBACK;
```

### Reading the Output

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'john@example.com';
```

**Without index:**
```
Seq Scan on users  (cost=0.00..234567.00 rows=1 width=72)
                   (actual time=8234.1..8234.1 rows=1 loops=1)
  Filter: (email = 'john@example.com')
  Rows Removed by Filter: 9999999
Execution Time: 8234.2 ms
```

**With index:**
```
Index Scan using idx_users_email on users
  (cost=0.56..8.58 rows=1 width=72)
  (actual time=0.089..0.091 rows=1 loops=1)
  Index Cond: (email = 'john@example.com')
Execution Time: 0.2 ms
```

### Field Meanings

| Field | Meaning |
|---|---|
| `cost=X..Y` | Planner cost units (not ms). X = startup, Y = total. 1 unit ≈ 1 sequential page read |
| `rows=N` | **Estimated** rows (planner guess from statistics) |
| `actual time=X..Y` | Real milliseconds (startup..total). ANALYZE only |
| `actual rows=N` | Rows actually returned. ANALYZE only |
| `loops=N` | How many times this node ran |
| `width=N` | Average row width in bytes |

### Types of Scans

| Scan Type | When used | Notes |
|---|---|---|
| Seq Scan | No index or large row fraction | Bad on large tables |
| Index Scan | Index + small result set | Good |
| Bitmap Heap Scan | Index + medium result set | Middle ground |
| Index Only Scan | All needed columns in the index | Best — never reads heap |

**Index Only Scan** — use `INCLUDE` to make a covering index:
```sql
CREATE INDEX idx_users_email_id ON users(email) INCLUDE (id, name);
-- SELECT id, name WHERE email = '...' → never touches the heap
```

### Join Algorithms

| Join Type | How | Best for |
|---|---|---|
| Nested Loop | Outer row → search inner | Small outer + indexed inner |
| Hash Join | Hash smaller table, probe with larger | Medium sets |
| Merge Join | Walk both sorted sets together | Large sorted/indexed sets |

**Danger pattern — Nested Loop + inner Seq Scan:**
```
Nested Loop
  Seq Scan on users  (rows=1000)
  Seq Scan on orders  (rows=50 loops=1000)  ← runs 1000 times!
```
Fix: index the join column on the inner table.

### Red Flags in EXPLAIN Output
1. `Seq Scan` on a large table → likely missing index
2. `rows=1` estimated but `actual rows=500000` → stale stats, run `ANALYZE`
3. `Nested Loop` with `Seq Scan` as inner → catastrophic if inner runs many times
4. One node has disproportionately high cost → that's your bottleneck

---

## Part 3 — The N+1 Problem [CRITICAL]

### What It Is
1 query to fetch N items + N queries for related data = N+1 total queries.

- 100 users → 101 queries
- 10,000 users → 10,001 queries
- Each query has ~0.1–0.5ms overhead → 10,000 queries = 1–5 seconds overhead alone

### Where It Comes From — ORM Lazy Loading

ORM = Object-Relational Mapper (library mapping DB tables to code objects: Django, Rails ActiveRecord, SQLAlchemy, etc.)

ORMs default to **lazy loading**: related data only fetched when accessed in code.

```python
# Django — N+1:
users = User.objects.all()          # 1 query
for user in users:
    print(user.posts.count())       # 1 query PER USER

# For 1000 users: 1001 queries
```

**In query logs:**
```
SELECT * FROM users
SELECT * FROM posts WHERE user_id = 1
SELECT * FROM posts WHERE user_id = 2
SELECT * FROM posts WHERE user_id = 3
... (997 more)
```
Same query template, different WHERE value = N+1 signature.

### Why It Goes Undetected Until Production

- Dev DB has 50 rows → 51 queries × 0.2ms = 10ms. Nobody notices.
- Production has 50,000 rows → 50,001 queries × 0.2ms = 10 seconds. Page timeout.

Code is identical. Data size changed. **N+1 is a scaling time-bomb.**

> **Real case: GitLab — 2018 onward**
> Found 5,000+ N+1 query occurrences. Some pages issued 400–600 queries per load.
> Root cause: Rails lazy loading + years of fast feature growth with no query auditing.
> Fix: Built an N+1 detector into their CI pipeline — new code that introduces N+1 queries fails the build.

### How to Fix N+1

**Fix 1: Eager loading (prefetch — 2 queries)**
```python
# Django: prefetch_related (for reverse FK / M2M)
users = User.objects.prefetch_related('posts').all()
# Query 1: SELECT * FROM users
# Query 2: SELECT * FROM posts WHERE user_id IN (1, 2, ..., 1000)
# No extra queries in the loop
```

```python
# Django: select_related (for forward FK — creates a JOIN)
posts = Post.objects.select_related('author').all()
# 1 query: SELECT posts.*, users.* FROM posts JOIN users ON posts.author_id = users.id
```

**Fix 2: Raw JOIN**
```sql
SELECT users.id, users.name, posts.id AS post_id, posts.title
FROM users
LEFT JOIN posts ON posts.user_id = users.id;
-- 1 query. LEFT JOIN = include users with no posts (their post columns are NULL)
```

**Fix 3: Batch IN clause**
```sql
-- Step 1: get user IDs
SELECT id FROM users WHERE active = true;
-- Returns: [1, 2, ..., 1000]

-- Step 2: one query for all related data
SELECT * FROM posts WHERE user_id IN (1, 2, ..., 1000);
-- Combine in app code
-- Total: 2 queries
```
This is what `prefetch_related` does internally.

### Detecting N+1

| Tool | Language | What it shows |
|---|---|---|
| Django Debug Toolbar | Python/Django | Query count + time per request |
| bullet_gem | Ruby/Rails | Console warnings when N+1 detected |
| `log_min_duration_statement = 0` | PostgreSQL | Logs every query to log file |
| `pg_stat_statements` | PostgreSQL extension | Aggregates: "this template ran 10,000 times" |

**Quickest approach:**
```sql
SET log_min_duration_statement = 0;
-- Run your app, hit the slow endpoint
-- Check PostgreSQL logs for repeated query patterns
```

### Watch Out For

- Eager loading with very large sets → `IN (1, 2, ..., 100000)` has its own overhead. Paginate first.
- Joining multiple one-to-many relations in one JOIN → row multiplication. Use separate prefetch queries.

---

## Production Trade-offs Summary

**Indexes:**
- Every index slows down writes (INSERT/UPDATE/DELETE must update all indexes)
- Indexes consume disk space (sometimes more than the table)
- Index columns you filter/sort/join on; don't index everything

**EXPLAIN:**
- Without ANALYZE: plan only, not reality. Row estimates may be very wrong.
- After bulk loads, run `ANALYZE table_name` to refresh statistics

**N+1:**
- Invisible in dev, catastrophic in production
- Fix with eager loading, JOINs, or IN clauses
- Add query count assertions to tests

---

## Connections to Previous Sessions

- **D02 (Client-Server):** The chain is browser → API server → DB. Query inefficiency is invisible to HTTP until the request times out.
- **D05 (REST):** N+1 appears at serialization — when an endpoint loops over a list and renders related fields, each field can trigger a query.
- **D07 (Nginx):** Nginx → app server → DB. The DB is the bottleneck no amount of Nginx tuning can fix.
- **D08 (Load Balancing):** Adding app servers multiplies DB load if N+1 is present. Fix the query first.

---

## Quiz Questions and Answers

**Q1.** You have a `products` table with 8 million rows. You add `CREATE INDEX idx_products_name ON products(name)`. Then you run `SELECT * FROM products WHERE LOWER(name) = 'iphone'`. EXPLAIN ANALYZE still shows a Seq Scan taking 6 seconds. Why, and what do you do?

**A1.** The index on `name` stores original-case values. `LOWER(name)` is a computed expression — the index on `name` cannot serve a query on `LOWER(name)`. PostgreSQL falls back to a Seq Scan. Fix: drop the broken index and create a functional index: `CREATE INDEX CONCURRENTLY idx_products_name_lower ON products(LOWER(name));`. Now `WHERE LOWER(name) = 'iphone'` uses the index.

---

**Q2.** EXPLAIN ANALYZE output:
```
Nested Loop  (actual time=0.1..52000.0 rows=80000 loops=1)
  Seq Scan on users  (rows=2000 loops=1)
  Seq Scan on orders  (rows=40 loops=2000)
    Filter: (orders.user_id = users.id)
    Rows Removed by Filter: 59960
```
What is wrong, and what single change fixes it?

**A2.** The inner `Seq Scan on orders` runs 2,000 times (`loops=2000` — once per user). Each run scans the entire orders table to find the matching ones. This is a missing index on the join column. Fix: `CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);`. The inner Seq Scan becomes an Index Scan running once per user at ~0.1ms instead of 26ms. Total: 200ms instead of 52 seconds.

---

**Q3.** `GET /api/posts` returns 500 blog posts with their author names. 80ms in staging (50 authors in DB), 12 seconds in production (50,000 authors in DB). Same code. What is the likely cause and fix?

**A3.** N+1 query problem. The endpoint fetches 500 posts (1 query), then for each post accesses `post.author.name`, triggering `SELECT * FROM users WHERE id = ?` per post — up to 500 individual lookups. In staging the users table is tiny so each is instant. In production the table has 50,000 rows; even at 0.5ms per query, 500 × 0.5ms = 250ms in overhead alone, plus connection overhead and possible missing index. Fix: eager-load authors with a JOIN (`select_related` in Django, `includes` in Rails) reducing 501 queries to 1. Also confirm `users.id` is indexed (it should be — it's the primary key, which is always indexed automatically).

---

---

## Follow-up Q&A From This Session

### Q: What is the difference between SQL and NoSQL?

**SQL** = databases that store data in tables (rows + columns). Every row has the same structure. Relationships between tables are explicit via foreign keys. Examples: PostgreSQL, MySQL, SQLite.

**NoSQL** = databases that don't use the table/row model. Several types:

| Type | Example | Stores data as |
|---|---|---|
| Document | MongoDB | JSON-like documents |
| Key-Value | Redis | key → value |
| Column-family | Cassandra | columns grouped by access pattern |
| Graph | Neo4j | nodes and edges |

MongoDB equivalent of a users+orders SQL schema embeds orders inside the user document — no separate table, no joins.

**One-line difference:** SQL enforces a fixed schema and strict relationships. NoSQL is flexible — documents can have different fields, no join requirement.

---

### Q: If I query `WHERE id = 20` and 20 is an internal node in the tree, does it find the row there?

No. PostgreSQL uses a **B+ Tree** (B-plus Tree), not a pure B-Tree. Critical distinction:

- **Internal nodes** = routing only. Store keys as separators to navigate. No ctid, no row pointer.
- **Leaf nodes** = ALL keys with their ctid. Every single queryable value lives here.

So `20` appears in an internal node as a separator **and also** in a leaf node with its ctid. The internal node just says "go left if ≤ 20." The actual pointer to the row is only in the leaf.

Query for `WHERE id = 20`:
```
Root [30]: 20 < 30 → go left
Internal [20]: route to left child
Leaf [20, 25]: found key 20 → ctid → fetch row from heap
```

**Why B+ Tree?** Leaf nodes are linked in a chain: `[10] ↔ [20,25] ↔ [40] ↔ [50,60]`. Range queries (`WHERE id BETWEEN 20 AND 50`) find the start leaf then walk the chain — no need to go back up the tree. Internal nodes having no ctids is what makes this chain possible.

---

### Q: What does `INCLUDE` actually do inside the tree?

`INCLUDE` columns are stored **only in leaf nodes**, invisible to internal nodes.

```sql
CREATE INDEX idx_users_email_id ON users(email) INCLUDE (id, name);
```

**Without INCLUDE — leaf stores:**
```
['bob@gmail.com'  →  ctid(page 7, slot 1)]
```
Query for `SELECT id, name WHERE email = 'bob@...'` → must follow ctid to heap to get id and name.

**With INCLUDE — leaf stores:**
```
['bob@gmail.com'  →  id=2, name='Bob'  →  ctid(page 7, slot 1)]
```
id and name are already in the leaf. Never touch the heap. This is an **Index Only Scan**.

Internal nodes still only see `email` for routing — they have no idea `id` and `name` exist. INCLUDE is purely "carry this extra data in leaf nodes to avoid a heap trip."

**Limitations:** INCLUDE columns cannot be used for `WHERE`, `ORDER BY`, or index navigation — only for fetching.

**Trade-off:** Leaf nodes get fatter (more disk space). Worth it only for hot, frequent queries that select a small fixed set of columns.

---

### Q: Explain join algorithms with proper examples

Three strategies. Setup: `users` (1,000 rows) joined to `orders` (50,000 rows) on `orders.user_id = users.id`.

**Nested Loop:**
```
for each of 1,000 users:
    search orders WHERE user_id = this_user.id
```
- Without index on `orders.user_id`: inner Seq Scan runs 1,000 times × 50,000 rows = 50M comparisons. O(n × m). Catastrophic.
- With index: inner Index Scan runs 1,000 times × ~23 comparisons = 23,000. Fast.
- Best when: small outer table + index on inner join column.

**Hash Join:**
```
Step 1 — build: read 1,000 users → build hash map { id → name }
Step 2 — probe: read 50,000 orders, look up each user_id in hash map → O(1) per lookup
```
Total: O(n + m). No index needed. The hash map IS the lookup structure.
Downside: hash map lives in memory. If `work_mem` too small → spills to disk → slow.

**Merge Join:**
```
users sorted by id:        1, 2, 3, 4...
orders sorted by user_id:  1, 1, 1, 2, 3, 3...

Walk both with two pointers, emit matches, advance the pointer with the smaller value.
```
O(n + m). Requires both sides sorted (or indexed). Never re-reads anything.

**The dangerous pattern:**
```
Nested Loop
  Seq Scan on users   (rows=1000, loops=1)
  Seq Scan on orders  (rows=50,   loops=1000)  ← runs 1000 times!
    Rows Removed by Filter: 49950
```
`rows=50` is per loop — total work is 50,000 × 1,000 = 50 million rows examined.
Fix: `CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);`

| Situation | PostgreSQL picks |
|---|---|
| Small outer + index on inner | Nested Loop |
| Medium tables, no index | Hash Join |
| Both sides sorted/indexed | Merge Join |
| Inner large + no index | Nested Loop + Seq Scan = disaster |

---

### Q: Is SQL more used in production or NoSQL?

SQL is the default. Most companies use PostgreSQL or MySQL for their core business data (users, payments, orders, transactions) because of ACID guarantees, decades of tooling, and structured data needs.

NoSQL is used **alongside** SQL, not instead of it:
- Redis → caching, sessions
- MongoDB → flexible/unstructured data
- Cassandra → massive write throughput
- Elasticsearch → full-text search

Typical production stack:
```
App → PostgreSQL (primary DB)
    → Redis       (cache + sessions)
    → Elasticsearch (search)
```

Most companies reach for Postgres first and add NoSQL only when they hit a specific problem Postgres can't solve well. D18 covers the full decision framework.

---

### Q: What is ORM lazy loading and why is it dangerous?

**ORM (Object-Relational Mapper)** = library that maps DB tables to code objects so you work with objects instead of raw SQL. Examples: Django, Rails ActiveRecord, SQLAlchemy.

**Lazy loading** = the ORM doesn't fetch related data until you actually access it in code.

```python
user = User.objects.get(id=1)
# SQL fires: SELECT * FROM users WHERE id = 1
# Posts? Not fetched yet.

user.posts.all()
# NOW SQL fires: SELECT * FROM posts WHERE user_id = 1
```

The danger: in a loop, each access silently fires a new query:
```python
users = User.objects.all()             # 1 query
for user in users:
    print(user.posts.count())          # 1 query PER USER — invisible in code
```

This looks like reading a variable. The DB call is completely hidden. A developer writing `user.posts.count()` in a loop has no idea they're firing 1,000 queries. It's harmless with 50 rows in dev, catastrophic with 50,000 rows in production. **N+1 is a scaling time-bomb.**

---

### Q: How to properly fix N+1 (from scratch)

**Fix 1 — Raw JOIN (pure SQL, 1 query):**
```sql
SELECT users.name, COUNT(posts.id) AS post_count
FROM users
LEFT JOIN posts ON posts.user_id = users.id
GROUP BY users.id, users.name;
```
JOIN merges both tables, COUNT+GROUP BY collapses to one row per user. `LEFT JOIN` includes users with zero posts.

**Fix 2 — Eager loading (ORM, 2 queries):**
```python
# prefetch_related — reverse FK (user → many posts)
users = User.objects.prefetch_related('posts').all()
# Query 1: SELECT * FROM users
# Query 2: SELECT * FROM posts WHERE user_id IN (1,2,...,1000)
# Loop accesses already-loaded data — no extra queries

# select_related — forward FK (post → its one author), does a JOIN
posts = Post.objects.select_related('author').all()
# 1 query: SELECT posts.*, users.* FROM posts JOIN users ON posts.author_id = users.id
```
Rule: going forward (one author) → `select_related`. Going backward (many posts) → `prefetch_related`.

**Fix 3 — Manual batch IN (no ORM):**
```python
users = db.query("SELECT id, name FROM users")
user_ids = [u.id for u in users]
posts = db.query(f"SELECT user_id, COUNT(*) as count FROM posts WHERE user_id IN ({','.join(map(str, user_ids))}) GROUP BY user_id")
post_counts = {p.user_id: p.count for p in posts}
# 2 queries total — this is what prefetch_related does internally
```

**When to use which:**
| Situation | Fix |
|---|---|
| Writing raw SQL | JOIN |
| Django/Rails ORM | `select_related` or `prefetch_related` |
| No ORM | Manual batch IN clause |
| 100,000+ related rows | Paginate first, then eager load |

**Watch out:** `IN (1..100000)` with very large sets has its own overhead — paginate first.

---

## Key Terms Introduced This Session

| Term | Definition |
|---|---|
| Seq Scan | Reading every row sequentially — no index used |
| Index Scan | Using an index to jump directly to matching rows |
| Index Only Scan | All needed columns in the index — heap never accessed |
| Bitmap Heap Scan | Index used first to build a bitmap of pages to read, then heap accessed |
| B-Tree | Balanced Tree — the default index structure in PostgreSQL |
| ctid | Physical row location (page_number, slot_number) |
| EXPLAIN | Shows the query plan PostgreSQL intends to use (doesn't run the query) |
| EXPLAIN ANALYZE | Runs the query and shows plan + actual timing/row counts |
| cost units | Planner's internal units — not milliseconds. 1 unit ≈ 1 sequential disk page read |
| N+1 problem | 1 query to get a list + N queries for related data = N+1 total |
| ORM | Object-Relational Mapper — library mapping DB tables to code objects |
| Lazy loading | ORM fetches related data only when accessed (default — causes N+1) |
| Eager loading | ORM fetches related data upfront in a single query |
| CONCURRENTLY | Index creation mode that doesn't lock the table during build |
| Functional index | Index on a computed expression, e.g. `LOWER(email)` |
| Nested Loop | Join algorithm: for each outer row, search inner table |
| Hash Join | Join algorithm: hash smaller table, probe with larger |
| FK (Foreign Key) | Column referencing the primary key of another table |
| Primary key | Column uniquely identifying each row — always indexed automatically |
