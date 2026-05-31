# D12 — API Versioning: How to Evolve APIs Without Breaking Clients

---

## Why It Matters

You can't update all consumers the same moment you update your server. They're independent systems deployed independently. API versioning gives clients a runway to migrate while your server keeps moving forward.

**Real case: Twitter — 2013**
- What happened: Twitter deprecated REST API v1, set a hard shutdown date (June 11, 2013), and killed it. Thousands of third-party apps went dark overnight.
- Root cause: Short migration window and no graceful strategy. The sunset was a cliff, not a slope. No programmatic deprecation signals — just blog posts.
- Cost/impact: Large loss of third-party developer ecosystem. Many small apps never migrated and simply died. Developers still cite this as a cautionary tale 10 years later.
- What would have prevented it: Longer runway + Sunset headers so clients could *detect programmatically* they were on a dying version.

---

## Two Classes of Changes

| Change type | Example | Safe without new version? |
|---|---|---|
| Non-breaking (additive) | Adding a new response field | Yes — clients ignore unknown fields |
| Non-breaking | Adding new optional query param | Yes |
| Non-breaking | Adding a new endpoint | Yes |
| **Breaking** | Renaming a field (`user_id` → `id`) | **No** |
| **Breaking** | Removing a field or endpoint | **No** |
| **Breaking** | Changing a field type (string → int) | **No** |
| **Breaking** | Changing auth mechanism | **No** |

**[CRITICAL]** Most common mistake: engineers introduce breaking changes thinking they're "just a cleanup" and don't version — then wake up to 500 errors from clients reading a field that no longer exists.

---

## The Four Versioning Strategies

### 1. URL Path Versioning [IMPORTANT]

```
GET /v1/users/123
GET /v2/users/123
```

**How it works internally:** Router inspects the path segment and dispatches to a versioned handler.

```python
@app.route("/v1/users/<int:user_id>")
def get_user_v1(user_id):
    return {"user_id": user_id, "name": "Aryan"}  # old schema

@app.route("/v2/users/<int:user_id>")
def get_user_v2(user_id):
    return {"id": user_id, "full_name": "Aryan"}  # new schema
```

**Advantages:** Easy to test in browser, cacheable at CDN level (URL = cache key), easy to route at Nginx/API Gateway level (can point `/v1/` to old deployment entirely)

**Disadvantages:** Technically not pure REST (URL should identify resource, not representation). If client hardcodes `/v1/` everywhere, migration touches every API call.

**In production:** Stripe, GitHub, Google APIs all use this. Most common in practice.

---

### 2. Header Versioning [IMPORTANT]

```
GET /users/123
Accept-Version: v1
```

**How it works internally:**

```python
@app.route("/users/<int:user_id>")
def get_user(user_id):
    version = request.headers.get("Accept-Version", "v2")
    if version == "v1":
        return {"user_id": user_id, "name": "Aryan"}
    return {"id": user_id, "full_name": "Aryan"}
```

**[CRITICAL] The Vary header trap:** You MUST include this in your response:
```
Vary: Accept-Version
```
This tells CDNs/proxies: "cache separately per Accept-Version header value." Without it, a v1 client gets a v2 cached response — a silent, hard-to-debug production bug.

**Advantages:** Clean URLs, more RESTful conceptually
**Disadvantages:** Can't test in browser, CDN caching requires careful Vary header handling

---

### 3. Query Parameter Versioning [GOOD TO KNOW]

```
GET /users/123?version=1
```

Less common. Same internals as header versioning but version is in query string. Version shows up in URLs, gets pasted/logged everywhere. Same CDN caching issue exists. Mostly seen in internal APIs built quickly.

---

### 4. Content Negotiation / Media Type Versioning [GOOD TO KNOW]

```
GET /users/123
Accept: application/vnd.myapp.v1+json
```

(`vnd` = vendor, the version is embedded in the media type in the `Accept` header)

Most technically correct per HTTP spec. Almost nobody uses it in practice for public APIs — too verbose and unfamiliar to most developers.

---

## Strategies Comparison

| Strategy | Where version lives | Cacheable easily | Testable in browser | Used in practice |
|---|---|---|---|---|
| URL path | `/v2/users/123` | Yes | Yes | Most common |
| Header | `Accept-Version: v2` | Only with `Vary` | No | Common |
| Query param | `?version=2` | With care | Yes | Less common |
| Media type | `Accept: vnd.app.v2+json` | Only with `Vary` | No | Rare |

**One-line answer:** URL path versioning wins in practice — visible, cacheable, loggable — even if not the purest REST approach.

---

## Deprecation Lifecycle [IMPORTANT]

```
New version released
        |
[BOTH versions live]  ← Migration window (ideally 6–12 months)
        |
Deprecation announced
        |
[Deprecated version]  ← Add Sunset headers HERE
        |
Sunset date reached
        |
[Old version removed]
```

### Sunset Headers — Machine-Readable Deprecation

`Sunset` is a standard HTTP response header — like `Content-Type` or `Vary` — that tells clients when this endpoint will stop working. The API still returns 200 and works normally; the header is just attached to every response saying "enjoy it while it lasts."

Add to responses from deprecated endpoints:
```
Sunset: Sat, 01 Jun 2026 00:00:00 GMT
Deprecation: true
Link: <https://api.myapp.com/v2/users>; rel="successor-version"
```

A client can check it programmatically:
```python
response = requests.get("/v1/users/123")

if "Sunset" in response.headers:
    print(f"WARNING: This endpoint dies on {response.headers['Sunset']}")
```

So even if no engineer on the client team read the announcement email, their monitoring catches it and pages someone.

**Analogy:** It's like a food expiry date printed on every carton — you don't need to remember when you bought it, it tells you right there every time you open the fridge.

Twitter's mistake: relied on blog posts and emails — easy to miss. Stripe's approach: every response from a deprecated version includes the date, and they have tooling that autodetects when an account is calling a deprecated version.

---

## Stripe's Versioning — The Gold Standard [IMPORTANT]

- Versions are dates: `2023-10-16`, `2024-04-10` (not `v1`, `v2`)
- When you create a Stripe account, you're **pinned to the current version** at signup
- Your account *is* the version — no version header needed in calls
- Breaking changes never affect existing accounts — only new signups get new behavior
- You opt into newer versions when ready via the dashboard

**Why dates instead of v1, v2?** With `v1`/`v2`, the number tells you nothing about what changed or when. With `2023-10-16`, you know exactly when your integration was frozen — you can look up Stripe's changelog for that exact date and see precisely what changed before and after your version.

**How pinning works in practice:**

```
Your account created: 2024-04-10
Your API call (2 years later): GET /v1/customers/123

Stripe internally: runs on 2026-01-01 logic
                        |
               transformation layer
                        |
               converts response back to 2024-04-10 schema
                        |
Your app receives: 2024-04-10 format ✓  nothing broke
```

**The transformation layer — how it works internally:**

Stripe has ONE unified codebase for business logic. But before sending a response, it passes through a serialization layer that reshapes the output based on the account's pinned version:

```
Modern server processes request
        |
Response in latest internal format
        |
[Transformation layer: account pinned to 2022-11-15?]
        |
Converts field names, restructures JSON to match 2022-11-15 schema
        |
Client gets response in their expected format
```

Every breaking change Stripe ever makes requires writing a new transformer. The transformation chain can be years long. They have engineers dedicated to this.

**When to use this vs URL versioning:**
- URL versioning (`/v1/`, `/v2/`) is right for 95% of APIs
- Stripe-style is only worth the investment when breakage is catastrophic — e.g. a payment API where one renamed field = real money lost, failed charges, enterprise clients with compliance review cycles who can't "just update their code"

---

## Code Example — URL Versioning in Express.js [IMPORTANT]

```javascript
// routes/v1/users.js
router.get('/:id', (req, res) => {
    res.json({ user_id: req.params.id, name: "Aryan" })
})

// routes/v2/users.js
router.get('/:id', (req, res) => {
    res.json({ id: req.params.id, full_name: "Aryan Gupta" })
})

// app.js
app.use('/v1/users', require('./routes/v1/users'))
app.use('/v2/users', require('./routes/v2/users'))
```

In production, this routing often happens at the **API Gateway** level (D10) — `/v1/*` goes to old deployment, `/v2/*` goes to new one entirely. No need to maintain both versions in the same codebase.

---

## Trade-offs & Common Mistakes [CRITICAL]

### When NOT to introduce a new version:
- Adding a new optional response field → additive, safe
- Adding a new optional query param → safe
- Adding a new endpoint → safe
- Performance fixes, bug fixes that don't change the contract

### Common production mistakes:

**1. No versioning at all** — "we'll tell clients to update" — works for internal services, catastrophic for public APIs or mobile apps (you cannot force-update an app on someone's phone)

**2. Premature cleanup** — removing a deprecated version because logs show "no one is calling it" — logs might miss batch jobs running once a week, or enterprise clients running on-prem integrations that never show up in your observability

**3. Forgetting the Vary header** — with header versioning; clients get wrong cached version silently

**4. Never removing old versions** — versions are a liability; every version maintained is more test surface, more security patches, more cognitive load; you need a real sunset process

**5. Over-versioning** — creating new versions for every change, including non-breaking ones → version sprawl where you maintain v1–v6 simultaneously with no one daring to delete anything

---

## Connection to Earlier Sessions

- **D05 (REST API):** REST defines resources via URLs, methods, status codes. Versioning is not specified by REST — it's your design decision.
- **D09 (Auth):** When you version an API, auth tokens/JWTs often need to be version-aware. A v1 token structure might not match v2's expected claims.
- **D10 (API Gateway):** API Gateways handle version routing in production. You can canary-release v2 to 5% of traffic while v1 handles 95% — the Gateway splits that traffic.

---

## Quiz & Answers

**Q1.** You add a new optional field `middle_name` to your `GET /v1/users/:id` response. A client that doesn't know about it calls the endpoint. Does this break them? Do you need a new version?

**A1.** No, it does not break them. Adding a new field is a non-breaking (additive) change. Well-written clients follow the robustness principle and ignore unknown fields. You do NOT need a new version.

---

**Q2.** You're using header versioning (`Accept-Version: v2`). Your response goes through a CDN. What must you include in your response to prevent v1 clients from getting a v2 cached response?

**A2.** You must include `Vary: Accept-Version` in the response header. This tells the CDN to cache separately per value of the `Accept-Version` request header. Without it, the CDN serves one cached response to everyone regardless of which version they requested.

---

**Q3.** You shut down v1 of your API. Three months later a large enterprise client's integration breaks — they say they never got any warning. What should you have done during the deprecation period to give them a *machine-readable* warning?

**A3.** You should have added `Sunset` and `Deprecation` response headers to v1 responses. `Sunset: <date>` tells clients programmatically when the endpoint dies. A monitoring tool on their side can detect this header and alert their team. Relying only on emails/blog posts misses clients who never read them.
