# D05 — REST API Design: Methods, Status Codes, Idempotency

**Date:** 2026-05-23

---

## The One-Line Summary

REST is just a style guide for how to use HTTP correctly — verb = action, URL = resource, status code = outcome.

---

## Why It Matters

HTTP gives you verbs, URLs, and status codes. REST says: *use them in a sane, predictable way.* Before REST, people were doing things like:
- `POST /getUser` — using POST to read
- `GET /deleteUser?id=5` — using GET to mutate (dangerous: browsers prefetch GET links)
- Returning `200 OK` with `{"error": "user not found"}` in the body — lying with status codes

REST is the industry agreeing on a convention so APIs are predictable. It's not a protocol, library, or framework. It's a **style guide for HTTP**.

---

## Resources

A **resource** is a thing your server manages (user, order, product). The URL identifies *what* you're acting on. The HTTP method identifies *what action* you're taking.

```
GET    /users/42      → read user #42
POST   /users         → create a new user
PUT    /users/42      → replace user #42 entirely
PATCH  /users/42      → update only specific fields of user #42
DELETE /users/42      → delete user #42
```

---

## HTTP Methods

| Method | Meaning | Has body? | Safe? | Idempotent? |
|--------|---------|-----------|-------|-------------|
| GET | Read a resource | No | Yes | Yes |
| POST | Create a new resource | Yes | No | No |
| PUT | Replace resource entirely | Yes | No | Yes |
| PATCH | Partially update resource | Yes | No | Sometimes |
| DELETE | Remove a resource | No | No | Yes |

**Safe** = doesn't change server state (GET is safe — reading changes nothing).  
**Idempotent** = calling it N times = same result as calling it once (defined below).

---

## Status Codes

| Range | Category |
|-------|----------|
| 2xx | Success |
| 3xx | Redirect |
| 4xx | Client error (you did something wrong) |
| 5xx | Server error (server broke) |

**Ones you'll use constantly:**

| Code | Name | When |
|------|------|------|
| 200 | OK | GET/PATCH/DELETE worked |
| 201 | Created | POST succeeded, new resource created |
| 204 | No Content | Success, nothing to return (common for DELETE) |
| 301 | Moved Permanently | URL changed forever |
| 400 | Bad Request | Malformed JSON, missing required field |
| 401 | Unauthorized | Not authenticated (server doesn't know who you are) |
| 403 | Forbidden | Authenticated but not permitted |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource already exists or state conflict |
| 422 | Unprocessable Entity | Syntax valid but semantic error (e.g., bad email format) |
| 429 | Too Many Requests | Rate limit hit |
| 500 | Internal Server Error | Generic server crash |
| 503 | Service Unavailable | Server overloaded or down |

**401 vs 403 trap:**
- 401 = "I don't know who you are" (not logged in)
- 403 = "I know who you are, you just can't do this" (not authorized)

---

## Idempotency — Deep Dive

**Idempotent** (eye-DEM-poh-tent): calling an operation once produces the same result as calling it N times in a row.

**Light switch analogy:**
- "Turn it OFF" = idempotent. Do it 10 times — after the first, same result.
- "Toggle it" = NOT idempotent. Each call flips the state.

**In HTTP:**

- `GET /users/42` — idempotent. Read it 100 times, same user returned.
- `DELETE /users/42` — idempotent. First call: deletes, returns 204. Second call: already gone, returns 404. But **state of the world is identical** — user doesn't exist either way.
- `POST /users` — NOT idempotent. Each call creates a *new* user. Five calls = five users.
- `PUT /users/42` — idempotent. You're saying "make user 42 exactly this." Call it 10 times, same result.

### Why it matters in production

Networks fail (D01). A client sends a DELETE and the response is dropped — it doesn't know if it succeeded. Should it retry?

- **Idempotent operation:** retry is safe. Worst case = 404 (already done). No harm.
- **Non-idempotent (POST):** retry = duplicate. In payments, this means charging the user twice. Real bug.

**Production fix for POST:** the client generates a unique **idempotency key** (a UUID — Universally Unique Identifier) and sends it in a header:

```
POST /payments
Idempotency-Key: f47ac10b-58cc-4372-a567-0e02b2c3d479
```

The server stores "this key → result already computed." On retry, it returns the cached result instead of processing again. Stripe uses this exact pattern.

---

## How the Server Routes a REST Request (Internal Flow)

Connecting back to D02 (request-response) and D04 (TLS):

```
Client                              Server
  |                                   |
  |  GET /users/42 HTTP/1.1           |
  |  Host: api.example.com            |
  |  Authorization: Bearer <token>    |
  |---------------------------------->|
  |                                   | 1. TCP connection accepted (D01)
  |                                   | 2. TLS decrypted (D04)
  |                                   | 3. HTTP parsed → verb=GET, path=/users/42
  |                                   | 4. Router matches (verb + path) → UserController
  |                                   | 5. Auth middleware checks Bearer token
  |                                   | 6. Controller queries DB
  |                                   | 7. Result serialized to JSON
  |                                   |
  |  HTTP/1.1 200 OK                  |
  |  Content-Type: application/json   |
  |  {"id":42,"name":"Aryan"}         |
  |<----------------------------------|
```

**Key point:** verb + path together form the route. Same path, different verb = different handler:
- `GET /users/42` → read handler
- `DELETE /users/42` → delete handler

---

## Code Example (FastAPI / Python)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

users = {1: {"id": 1, "name": "Aryan", "email": "a@x.com"}}

class UserUpdate(BaseModel):
    name: str | None = None
    email: str | None = None

# GET — safe, idempotent
@app.get("/users/{user_id}")
def get_user(user_id: int):
    if user_id not in users:
        raise HTTPException(status_code=404, detail="User not found")
    return users[user_id]  # 200 OK implicitly

# POST — not idempotent, creates new resource
@app.post("/users", status_code=201)
def create_user(user: dict):
    new_id = max(users.keys()) + 1
    users[new_id] = {"id": new_id, **user}
    return users[new_id]

# PATCH — partial update
@app.patch("/users/{user_id}")
def update_user(user_id: int, update: UserUpdate):
    if user_id not in users:
        raise HTTPException(status_code=404)
    if update.name:
        users[user_id]["name"] = update.name
    if update.email:
        users[user_id]["email"] = update.email
    return users[user_id]

# DELETE — idempotent
@app.delete("/users/{user_id}", status_code=204)
def delete_user(user_id: int):
    if user_id not in users:
        raise HTTPException(status_code=404)
    del users[user_id]
    # 204 = success, no body
```

---

## Trade-offs & When NOT to Use REST

| Situation | REST? | Better alternative |
|-----------|-------|-------------------|
| Public CRUD API | Yes | — |
| Real-time bidirectional data | No | WebSockets (D10) |
| Mobile app needing nested data in one shot | Sometimes | GraphQL |
| Internal microservice calls (performance critical) | Maybe | gRPC (D29) |
| Action-oriented ops (run job, send email) | Awkward | RPC or message queue |

**REST bends awkwardly for actions that aren't resources.** E.g., `POST /users/42/send-password-reset` — that's not a resource, it's a command. REST doesn't model commands well.

---

## Follow-up Q&A from Session

**Q: So REST API is just how to use HTTP? That's all?**

A: Pretty much yes — but that "how" is doing a lot of work. HTTP gives you verbs, URLs, status codes. REST says: use verbs to mean actions, use URLs to identify resources, use status codes correctly. It's not a new protocol or library — it's a **style guide**. When someone says "we have a REST API," they mean "we use HTTP in the conventional, predictable way." The one place it has real teeth: idempotency is a hard constraint with real consequences (double payments, duplicates) — that's where the convention matters most.

---

## Quiz

**Q1.** You send `DELETE /orders/99` and get a network timeout. Is it safe to retry? Why?

**A1.** Yes — DELETE is idempotent. If the order was already deleted, you get 404. The state of the world is identical either way (order doesn't exist). No harm in retrying.

---

**Q2.** What's the difference between 401 and 403?

**A2.** 401 = server doesn't know who you are (not authenticated, no valid token). 403 = server knows who you are but you don't have permission (not authorized for this action).

---

**Q3.** A client calls `POST /payments`, network drops before the response. Client retries. What goes wrong, and how do production systems prevent it?

**A3.** POST is not idempotent — each call creates a new payment. Retry = double charge. Fix: client sends a unique `Idempotency-Key` header (UUID). Server stores "key → result" and returns the cached response on retry instead of processing again.
