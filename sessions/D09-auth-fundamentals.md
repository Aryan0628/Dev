# D09 — Auth Fundamentals: Session vs JWT vs OAuth 2.0

**Date:** 2026-05-27
**Phase:** 1 — Foundations

---

## Why It Matters

HTTP is stateless (D02) — every request is independent, server has zero memory of previous ones. But every real app needs to know: *who is making this request?* Auth is the mechanism that attaches identity to a stateless protocol.

---

## The Three Approaches

### 1. Session-based Auth

**Analogy:** Hotel key card — the card itself has no info. The front desk has a register: "card #4821 = Room 302 = Aryan." Every door scanner calls the front desk to look it up.

**Flow:**
```
Client                          Server
  |                               |
  |-- POST /login (user+pass) --> |
  |                               | creates session in Redis
  |                               | session_id = "a7f3b2"
  |                               | Redis: { "a7f3b2": { user_id: 42 } }
  |<-- Set-Cookie: sid=a7f3b2 --- |
  |                               |
  |-- GET /dashboard              |
  |   Cookie: sid=a7f3b2 -------> |
  |                               | lookup "a7f3b2" in Redis → user_id=42
  |<-- 200 your dashboard ------- |
```

**Key insight:** Session lives on the **server**. Client only holds a meaningless random ID.

**Cookie flags:**
- `HttpOnly` — JavaScript on the page cannot read the cookie (blocks XSS — cross-site scripting — theft)
- `Secure` — only sent over HTTPS
- `SameSite=Strict` — only sent to same origin, blocks CSRF (cross-site request forgery — where another site tricks your browser into sending a request)

---

### 2. JWT (JSON Web Token)

**Analogy:** Laminated badge that says "Room 302, Aryan, expires midnight." Signed with the manager's stamp. Any door scanner verifies the stamp — no front desk call needed. Info travels WITH you.

**JWT = JSON Web Token** (JSON = JavaScript Object Notation; Token = self-contained credential)

**Structure:** `header.payload.signature` — three base64url-encoded parts, dot-separated

```
eyJhbGciOiJIUzI1NiJ9 . eyJ1c2VyX2lkIjo0Mn0 . SflKxwRJSMeKKF2QT4fw
      header               payload              signature
```

- **Header** — algorithm used: `{ "alg": "HS256", "typ": "JWT" }`
- **Payload** — claims: `{ "user_id": 42, "exp": 1717000000, "iat": 1716990000 }`
  - `exp` = expiry timestamp, `iat` = issued-at timestamp
- **Signature** — `HMAC_SHA256(header + "." + payload, SERVER_SECRET)`

**Flow:**
```
Client                          Server
  |                               |
  |-- POST /login (user+pass) --> |
  |                               | builds + signs JWT
  |<-- { token: "eyJ..." } ------ |
  |                               |
  |-- GET /dashboard              |
  |   Authorization: Bearer eyJ-> |
  |                               | recompute signature → match → valid
  |                               | decode payload → user_id=42 (no DB lookup)
  |<-- 200 your dashboard ------- |
```

**Key insight:** State lives on the **client**. Any server in your cluster can verify independently — no shared store needed.

**Signing algorithms:**
- **HS256** = HMAC-SHA256 (HMAC = Hash-based Message Authentication Code). Symmetric — same secret signs and verifies. One server only.
- **RS256** = RSA-SHA256 (RSA = Rivest-Shamir-Adleman). Asymmetric — private key signs, public key verifies. Used when multiple services need to verify tokens without sharing the secret.

**Critical:** The payload is NOT encrypted — only signed. Anyone can base64-decode it and read `user_id: 42`. Never put passwords, SSNs, or sensitive data in a JWT payload.

**localStorage** = browser API storing key-value pairs locally. Persists across tabs/refreshes. Unlike cookies, NOT automatically sent with requests — you must manually attach the token to headers.

---

### 3. OAuth 2.0

**OAuth = Open Authorization** (version 2.0)

**Key distinction:**
- **Authentication** = proving *who you are* ("I am Aryan")
- **Authorization** = proving *what you're allowed to access* ("I can read Aryan's calendar")

OAuth 2.0 is an **authorization framework**, but widely used for "Login with Google/GitHub." That specific use case is technically **OpenID Connect** (OIDC = OpenID Connect) — OAuth 2.0 + an identity layer. In practice, people call both "OAuth."

**Three parties:**
- **Resource Owner** = you (the user)
- **Client** = your app
- **Authorization Server** = Google, GitHub, etc.

**Authorization Code Flow (most common):**
```
User          Your App (Client)          Google (Auth Server)
  |                  |                          |
  |-- click          |                          |
  |  "Login w/Google"|                          |
  |                  |                          |
  |<-- redirect -----+-- GET /auth?             |
  |   to Google         client_id=xyz           |
  |   login page        redirect_uri=...        |
  |                     scope=email ----------->|
  |                                             |
  |-- logs in on Google ---------------------->|
  |                                             |
  |<-- redirect back with ?code=AUTH_CODE ------+
  |                  |                          |
  |                  |-- POST /token            |
  |                  |   code=AUTH_CODE         |
  |                  |   client_secret=... ---->|
  |                  |                          |
  |                  |<-- { access_token: ... } |
  |                  |                          |
  |                  |-- GET /userinfo -------->|
  |                  |<-- { email: "aryan@..." }|
```

**Why the two-step (code → token)?**
The `?code=AUTH_CODE` lands in the browser URL — visible in address bar, logs, browser history. If it were the access token directly, it'd be exposed. The code is:
- Single-use
- Short-lived (~60 seconds)
- Useless without `client_secret` (only your backend knows it)

The final token exchange is **server-to-server**, never visible to the browser.

**Scope** = what permissions you're requesting. `scope=email profile` → Google shows "this app wants your email and profile." Token is only valid for those scopes.

---

## Internals

### Session internals

```js
const session_id = crypto.randomBytes(32).toString('hex')
redis.set(session_id, JSON.stringify({ user_id: 42 }), 'EX', 86400)
// EX 86400 = expires in 86400 seconds = 24 hours
res.setHeader('Set-Cookie', `sid=${session_id}; HttpOnly; Secure; SameSite=Strict`)
```

On next request: read cookie → Redis lookup → get user record.

### JWT internals

```
header    = base64url({ "alg": "HS256", "typ": "JWT" })
payload   = base64url({ "user_id": 42, "exp": 1717000000, "iat": 1716990000 })
signature = HMAC_SHA256(header + "." + payload, SERVER_SECRET)
jwt       = header + "." + payload + "." + signature
```

Verification:
```
[header, payload, received_sig] = token.split(".")
expected_sig = HMAC_SHA256(header + "." + payload, SERVER_SECRET)
if expected_sig == received_sig → valid, decode payload
```

No DB lookup. Pure CPU math.

---

## Code Examples

### Session (Node.js / Express)

```js
const session = require('express-session')
const RedisStore = require('connect-redis')(session)

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { httpOnly: true, secure: true, maxAge: 86400000 }
}))

app.post('/login', async (req, res) => {
  const user = await db.findUser(req.body.email)
  if (!user || !bcrypt.compareSync(req.body.password, user.hash))
    return res.status(401).json({ error: 'Invalid credentials' })
  req.session.userId = user.id   // stored in Redis, cookie sent automatically
  res.json({ ok: true })
})
```

### JWT (Node.js)

```js
const jwt = require('jsonwebtoken')

// Issue token
app.post('/login', async (req, res) => {
  const user = await db.findUser(req.body.email)
  if (!user || !bcrypt.compareSync(req.body.password, user.hash))
    return res.status(401).json({ error: 'Invalid credentials' })
  const token = jwt.sign(
    { user_id: user.id },
    process.env.JWT_SECRET,
    { expiresIn: '1h' }
  )
  res.json({ token })
})

// Verify token
app.get('/dashboard', (req, res) => {
  const token = req.headers.authorization?.split(' ')[1] // "Bearer <token>"
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET)
    res.json({ userId: decoded.user_id })
  } catch (e) {
    res.status(401).json({ error: 'Invalid token' })
  }
})
```

---

## Comparison Table

| | **Session** | **JWT** | **OAuth 2.0** |
|---|---|---|---|
| State lives | Server (Redis/DB) | Client (token) | Client (token) |
| Scalability | Needs shared store across servers | Easy — any server verifies | Easy |
| Revocation | Instant — delete the session | Hard — valid until `exp` | Via refresh token rotation |
| Per-request cost | Redis lookup (~1ms) | CPU signature verify | Depends |
| Security risk | Session hijacking (stolen cookie) | Token theft (stolen JWT) | Misconfigured redirect_uri |
| Best for | Monoliths, traditional web apps | APIs, microservices, mobile | Third-party login / delegation |

---

## Trade-offs & When NOT To Use

**Sessions — don't use when:**
You have multiple servers (D08 — load balancing) with no shared Redis. User logs in on server 1, next request goes to server 3 → session not found → logged out. Sticky sessions (D08) can patch this but are fragile.

**JWT — don't use when:**
You need instant revocation. User gets banned → JWT still valid until `exp`. A "token blocklist" DB lookup on every request defeats the stateless benefit. For sensitive systems (banking, admin), sessions are safer.

**OAuth — don't use when:**
It's your own app with your own users. OAuth is for delegating access to a third party. "Login with Google" for your own employee portal is overkill and adds an external dependency.

---

## Connections to Previous Sessions

- **D02** — HTTP is stateless → root reason auth is non-trivial
- **D04** — TLS — the `Secure` cookie flag requires HTTPS (TLS); JWT signature uses the same asymmetric crypto concepts (RS256)
- **D05** — REST APIs are stateless → JWT fits better than sessions for REST APIs (no server-side state)
- **D07/D08** — Multiple Nginx servers → sessions need a shared Redis store, or use JWT to avoid the problem entirely

---

## Quiz

**Q1:** HTTP is stateless. Explain in one sentence why this makes authentication non-trivial.
**A:** Because each request carries no memory of previous ones, the server has no way to know "this is the same person who logged in" without the client proving identity on every request.

**Q2:** You issue a JWT. The user gets banned 5 minutes later. JWT expires in 55 minutes. What happens when they hit a protected route?
**A:** They get in. The signature is valid, expiry hasn't hit, server has no way to know they're banned without a blocklist. This is the core weakness of stateless tokens.

**Q3:** In OAuth's Authorization Code flow, why is there a two-step process (code first, then exchange for token) instead of redirecting back with the access token directly?
**A:** The code arrives via browser redirect — visible in URL, logs, browser history. The code alone is useless without `client_secret` (only your backend knows it). The token exchange is server-to-server, never exposed to the browser.

---

## Follow-up: The `state` Parameter — CSRF Attack Explained

### What CSRF Is (Without OAuth)

**CSRF = Cross-Site Request Forgery** — an attack where a malicious site triggers a request to another site using your browser's credentials.

Example: You're logged into `yourbank.com`. Attacker's page at `evil.com` has a hidden form that auto-submits `POST yourbank.com/transfer`. Your browser auto-attaches your bank session cookie. Bank sees a valid session and executes the transfer. You never clicked anything on the bank's site.

Root cause: the browser attaches cookies automatically to every request for that domain — even requests triggered by other sites.

---

### The OAuth CSRF Attack (Without `state`)

**Goal:** Attacker wants to link their Google account to the victim's `yourapp.com` account. If they succeed, they can use "Login with Google" with their own Google password and land inside the victim's account.

**Three people:** You (victim, logged into yourapp.com), Attacker, Google.

**Step 1 — Attacker starts their own login flow**
Attacker visits yourapp.com, clicks "Login with Google," logs into their own Google account. Google is about to redirect to:
```
https://yourapp.com/callback?code=ATTACKERS_CODE
```
Attacker stops here and copies this URL. Does not follow it.

**Step 2 — Attacker tricks you into visiting that URL**
Via phishing link, hidden `<img>` tag, iframe — anything that makes your browser send:
```
GET https://yourapp.com/callback?code=ATTACKERS_CODE
Cookie: sid=YOUR_SESSION  ← browser auto-attaches this
```

**Step 3 — yourapp.com processes it in your session**
Your backend sees a valid session (yours) and a valid `code`. It exchanges the code with Google — gets tokens for the attacker's Google account. Links that account to your yourapp.com account.

**Result:** Attacker can now log into your yourapp.com account using their own Google credentials.

---

### How `state` Blocks This

When you legitimately start a login flow:
1. Your server generates `state = "f9a2k7x"` (random)
2. Stores it in your server-side session: `session["state"] = "f9a2k7x"`
3. Sends it to Google in the redirect URL
4. Google returns it in the callback: `?code=...&state=f9a2k7x`

When your server gets the callback, first thing it does:
```
state in URL == state stored in session?
```

In the attack: attacker tricks your browser into visiting `/callback?code=ATTACKERS_CODE&state=ATTACKERS_STATE`. Your session has **no state stored** (you never started a login flow). Mismatch → request killed. Code never gets exchanged.

**Key insight:** Attacker can make your browser send requests — CSRF is real. But they cannot inject a value into your server-side session. `state` ties the callback to the specific session that initiated the flow.

---

## Follow-up: Full OAuth Flow — Browser + Server + Storage Simultaneously

### Step 1 — User Clicks "Login with Google"

```
BROWSER                         YOUR SERVER                     BROWSER STORAGE
──────────────────────────────  ──────────────────────────────  ──────────────────────
User clicks button.             GET /auth/google received.      Session cookie may
                                                                not exist yet.
                                Generates:
                                  state = "f9a2k7x" (random)

                                Saves to Redis:
                                  session["state"] = "f9a2k7x"

                                Sets cookie in response:        ← Set-Cookie: sid=SESSION_ID_123
                                  sid=SESSION_ID_123
                                                                SESSION COOKIE SAVED:
                                                                sid=SESSION_ID_123

                                Builds Google redirect URL:
                                  /auth?client_id=xyz
                                  &redirect_uri=yourapp.com/cb
                                  &scope=openid email
                                  &state=f9a2k7x
                                  &response_type=code

                                Sends 302 Redirect → Google
Browser follows 302.
Navigates to Google.
```

### Step 2 — User Is On Google's Page

```
BROWSER                         YOUR SERVER                     BROWSER STORAGE
──────────────────────────────  ──────────────────────────────  ──────────────────────
At accounts.google.com.         Doing nothing. Waiting.         Your app's cookie
Google shows login form.                                        (sid=...) still exists.
User types Google credentials.                                  NOT sent to Google
User clicks Sign In.                                            — different domain.

                                                                Google sets their own
                                                                cookies on google.com
                                                                (separate, unrelated)
```

Your app is completely out of this. Credentials go only to Google.

### Step 3 — Google Redirects Back

```
BROWSER                         YOUR SERVER                     BROWSER STORAGE
──────────────────────────────  ──────────────────────────────  ──────────────────────
Google sends 302:               GET /callback received:         Browser auto-attaches
Location: yourapp.com/callback    ?code=4/Pq7W91a...            session cookie:
  ?code=4/Pq7W91a...              &state=f9a2k7x                Cookie: sid=SESSION_ID_123
  &state=f9a2k7x                  Cookie: sid=SESSION_ID_123

Browser follows redirect.       Reads sid → Redis lookup
                                  → session["state"] = "f9a2k7x"

                                Compares:
                                  URL state:     "f9a2k7x" ✓
                                  Session state: "f9a2k7x" ✓
                                  MATCH → safe to continue

                                Deletes state from session
                                (single-use, discard after check)
```

### Step 4 — Server Exchanges Code for Token (Browser Not Involved)

```
BROWSER                         YOUR SERVER                     BROWSER STORAGE
──────────────────────────────  ──────────────────────────────  ──────────────────────
Spinner. Waiting.               POST google.com/token           Nothing changes.
Browser is idle.                  code=4/Pq7W91a...             Browser is idle.
                                  client_id=xyz
                                  client_secret=SECRET
                                  redirect_uri=yourapp.com/cb
                                  grant_type=authorization_code

                                Google verifies all three →
                                responds:
                                  {
                                    access_token: "ya29...",
                                    id_token: "eyJ...",
                                    refresh_token: "1//0g...",
                                    expires_in: 3600
                                  }

                                Decodes id_token (JWT):
                                  sub: "google-uid-42"
                                  email: "aryan@gmail.com"

                                Looks up/creates user in DB.
                                Saves to session:
                                  session["user_id"] = 42
                                  session["logged_in"] = true

                                refresh_token → stored in DB
                                NOT sent to browser ever.
```

### Step 5 — User Is Logged In

```
BROWSER                         YOUR SERVER                     BROWSER STORAGE
──────────────────────────────  ──────────────────────────────  ──────────────────────
Server sends 302 → /dashboard   302 Location: /dashboard        sid=SESSION_ID_123
Browser follows it.                                             now points to session
GET /dashboard                  Reads sid → user_id=42          with user_id=42 in it.
  Cookie: sid=SESSION_ID_123      logged_in=true
                                Renders dashboard.
Browser shows dashboard.
```

**After OAuth completes, your app uses its own session for all future requests — OAuth was only needed for the login step, not for every request.**

**One-line summary:** Your server generates `state`, hides it in your session, redirects you to Google — Google sends you back with a `code` and the same `state` — your server checks the `state` matches what it stored, then exchanges the `code` for tokens directly with Google (server-to-server, browser idle), marks the session as logged in, and redirects to the dashboard.
