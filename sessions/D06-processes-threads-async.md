# D06 — Processes vs Threads vs Async
### (Event loop, blocking vs non-blocking, everything connected)

**Date:** 2026-05-24

---

## One-Line Summary

The OS always creates the same TCP connection and FD — the difference is who waits for data: a dedicated blocked process, a dedicated blocked thread, or the OS kernel itself (via epoll) while one thread handles everything else.

---

## Why It Matters

When your server waits for a DB response (50ms), what does it do with the CPU?
- **Bad:** Sit idle, blocked, doing nothing
- **Good:** Handle 200 other requests while waiting

Processes, threads, and async are three different answers to this one problem.

---

## I/O is Slow — The Fundamental Problem

| Operation | Time | CPU cycles wasted |
|---|---|---|
| RAM access | 100 ns | ~300 |
| SSD read | 100 µs | ~300,000 |
| Network round trip | 0.5 ms | ~1,500,000 |
| Database query | 10–50 ms | ~30,000,000+ |

---

## Process

An independent program with its own memory space, FD table, and PID. Completely isolated — like a separate house.

```
Process A memory: [own stack] [own heap] [own globals]
Process B memory: [own stack] [own heap] [own globals]
↑ completely separate — A cannot read B's memory
```

- **Cost:** ~1MB RAM, ~5ms to spawn
- **Every process has at least 1 thread** — the main thread. A process cannot exist without at least one thread. The process is the container; the thread is what actually executes code.

---

## Thread

An execution unit inside a process. Multiple threads share the same memory.

```
Process:
  ├── Thread 1: [own stack] ─┐
  ├── Thread 2: [own stack] ─┼─ all share heap + globals
  └── Thread 3: [own stack] ─┘
```

- **Cost:** ~8KB per thread (just the stack)
- **Risk:** shared memory → race conditions, deadlocks

### Sockets belong to the process, NOT threads

Sockets are file descriptors (FDs) stored in the **process's FD table** — not owned by any thread. Any thread can access any FD. The stack is private to each thread; sockets are not.

```
Process:
  ├── FD table: [fd3, fd47, fd48...]  ← shared by ALL threads
  ├── Thread 1: [own stack] — can read/write any FD
  └── Thread 2: [own stack] — can read/write any FD
```

---

## GIL — Python's Threading Catch

**GIL** = Global Interpreter Lock. Only one thread can execute Python bytecode at a time, even on 8 CPU cores.

- Python is **not single-threaded** — it has real OS threads
- But threads are **concurrent, not parallel** for CPU work (GIL blocks them)
- For I/O: thread releases GIL while waiting → other threads can run
- **Node.js** is the language actually called single-threaded (1 JS thread, event loop only)

| | Python | Node.js |
|---|---|---|
| Threads? | Yes, real OS threads | No JS threads |
| CPU parallel? | No (GIL) | No (1 thread) |
| I/O concurrent? | Yes | Yes (event loop) |

---

## Thread States

A thread is always in one of three states:

```
RUNNING  → executing on CPU right now
READY    → wants to run, waiting for a free CPU core
BLOCKED  → waiting for I/O, timer, or lock — NOT on CPU
```

---

## Preemptive Scheduling — Why OS Forcefully Stops Threads

Without forcing, one greedy thread could hog the CPU forever. So the OS uses a **hardware timer interrupt** that fires every ~1–10ms:

```
1. Thread A is RUNNING
2. Timer fires → CPU jumps to OS interrupt handler (Thread A has no say)
3. OS saves Thread A's state (CPU registers, program counter, stack pointer)
4. OS picks Thread B from READY queue
5. OS loads Thread B's saved state → Thread B continues
```

This save + restore = **context switch** (~1–10µs).

**Timer only applies to RUNNING threads.** BLOCKED threads are invisible to the scheduler — not in the ready queue, not consuming CPU, timer doesn't touch them.

```
Thread BLOCKED → timer fires 100 times → thread invisible, 0% CPU ("timer off")
Data arrives → OS: BLOCKED → READY → RUNNING → timer applies again ("timer on")
```

---

## Blocking vs Non-Blocking I/O

**Blocking I/O:** Thread calls `read()` — OS puts it to sleep until data arrives. Thread does nothing.

```
Thread: read(fd47) → [BLOCKED... waiting...] → data arrives → continue
```

**Non-blocking I/O:** Thread asks OS "is data ready?" — if no, OS returns immediately. Thread does other work and checks again later.

**epoll** = OS syscall (system call — request to kernel) that lets one thread watch thousands of sockets simultaneously and get notified when any has data.

---

## Async / Event Loop

A **single thread** running a loop:

```
while True:
  1. Run all coroutines that are currently READY
  2. When none are ready → call epoll_wait() (thread BLOCKED, timer off)
  3. OS wakes thread: "these sockets have data"
  4. Mark those coroutines as READY
  5. Go back to step 1
```

**Coroutine** = a function that can pause itself at `await` and resume later without blocking the thread. Cooperative — it voluntarily yields control.

### What `await` actually does

```python
async def handle():
    user = await db.query("SELECT ...")
    # 1. Sends SQL bytes to socket (non-blocking, instant)
    # 2. Registers socket with epoll: "tell me when data arrives"
    # 3. Pauses this coroutine, hands control back to event loop
    # 4. Event loop runs other coroutines
    # 5. DB responds → epoll notifies event loop → this coroutine resumes here
    return user
```

No thread sleeping. OS watches the socket. Thread stays busy running other coroutines.

### Event loop flow with 3 concurrent requests

```
Timeline (single thread):

[run A]─await─[run B]─await─[run C]─await─[epoll_wait]─[resume B]─[resume A]─[resume C]
         ↑           ↑           ↑              ↑
     pauses A    pauses B    pauses C    OS watching all 3 sockets
```

All 3 complete in ~time of 1 DB query.

---

## The Three Concurrency Models

### Common ground — always happens first

```
Client connects → OS TCP handshake → OS creates fd47 in server's FD table
```

### Listening socket vs Connection socket

Before any client connects, server does at startup:
```
socket() → bind(port 443) → listen() → accept()
                                           ↑
                               blocks here (RUNNING → BLOCKED)
```

| | Listening socket | Connection socket |
|---|---|---|
| Example | fd 3 | fd 47 |
| Job | Wait for new clients | Talk to one specific client |
| Owner | Parent process | Child process / thread / coroutine |
| Lives | Forever (server running) | Until connection closes |

### Model 1 — Process per connection

```
fd47 arrives → parent forks child process (~1MB, ~5ms)
→ child gets copy of FD table (including fd47)
→ parent closes fd47, goes back to accept(fd3) → BLOCKED
→ child: read(fd47) → BLOCKED (timer off)
→ data arrives → BLOCKED → READY → RUNNING (timer on)
→ child processes request, responds, dies

1000 connections = 1000 processes = ~1GB RAM
```

### Model 2 — Thread per connection

```
fd47 arrives → server grabs thread from pool (~8KB)
→ thread: read(fd47) → BLOCKED (timer off)
→ data arrives → BLOCKED → READY → RUNNING (timer on, OS can preempt)
→ thread processes request, returns to pool

1000 connections = 1000 threads = ~8MB RAM
Risk: shared memory → race conditions
```

### Model 3 — Async / Event loop

```
fd47 arrives → add fd47 to epoll list
→ create coroutine_47, run until it hits await
→ await: register fd47 with epoll, pause coroutine_47
→ event loop runs other coroutines
→ nothing to run → epoll_wait() → thread BLOCKED (timer off)
→ data on fd47 → OS → BLOCKED → READY → RUNNING
→ event loop resumes coroutine_47 exactly where it paused

1000 connections = 1000 coroutines, 1 thread = ~KB total overhead
```

---

## Thread blocking vs Async event loop blocking — the key difference

Both are BLOCKED with timer off. The difference is what they're blocked ON:

| | Thread model | Async event loop |
|---|---|---|
| Blocked on | `read(fd47)` — one specific socket | `epoll_wait([all fds])` — all sockets at once |
| Threads sleeping | 1 per connection | 1 total (always) |
| Memory | 1000 × 8KB = ~8MB | 1 × 8KB = ~8KB |
| Who watches sockets | OS wakes each thread individually | OS wakes 1 thread, it handles all |

**Analogy:** Thread model = 1000 people each sleeping next to their own phone. Async = 1 person sleeping in front of a switchboard showing all 1000 phones.

---

## Trade-off Table

| | Processes | Threads | Async |
|---|---|---|---|
| Isolation | Full (separate memory) | None (shared memory) | None (same thread) |
| CPU parallelism | Yes (multiple cores) | Yes (except Python GIL) | No — single thread |
| Overhead | High (~1MB, ~5ms spawn) | Medium (~8KB stack) | Very low |
| Best for | CPU-heavy work, safety | Mixed I/O + CPU | High-concurrency I/O |
| Pitfalls | IPC complexity, slow spawn | Race conditions, deadlocks | Blocking code kills performance |

**One-line rule:** I/O-bound → async. CPU-bound → processes (or threads, not Python).

**When NOT to use async:**
- CPU-heavy work — one coroutine hogging the loop blocks everyone
- Accidentally calling blocking code inside async — freezes the entire event loop

```python
async def bad():
    time.sleep(5)           # blocks entire event loop for 5 seconds

async def good():
    await asyncio.sleep(5)  # only pauses this coroutine, loop stays free
```

---

## Everything Connected — Full Flow (D01–D06)

### How browser decides TCP vs QUIC

```
First visit to server:
→ Browser doesn't know if server supports QUIC
→ Defaults to TCP (always works)
→ TCP handshake → TLS handshake → HTTP/2 response includes:
   Alt-Svc: h3=":443"  ← "I support HTTP/3 on QUIC"
→ Browser caches this

Second visit:
→ Browser sees cached Alt-Svc → tries QUIC directly
→ Some browsers race TCP and QUIC simultaneously, use whichever wins
```

**Alt-Svc comes in the HTTP response** — not during TLS. Browser never destroys an active TCP connection mid-session. It switches to QUIC on the next visit.

### ALPN — how HTTP version is negotiated inside TLS

**ALPN** = Application-Layer Protocol Negotiation. Extension inside TLS handshake:

```
Client TLS ClientHello:
  ALPN: ["h2", "http/1.1"]   ← list of supported HTTP versions

Server TLS ServerHello:
  ALPN: "h2"                  ← server picks one
```

TCP handshake has no HTTP version info — it's just a raw pipe. HTTP version is decided inside TLS, before the first HTTP request. Fixed for the entire connection after that.

### How many TCP connections per HTTP version

Port never changes (80 or 443). What changes is how many connections the browser opens:

| Version | TCP connections | Requests per connection |
|---|---|---|
| HTTP/1.1 | Up to 6 per domain | 1 at a time (sequential) |
| HTTP/2 | 1 per domain | Many (multiplexed streams) |
| HTTP/3 | 1 QUIC connection | Many (independent QUIC streams) |

OS tells connections apart by 4-tuple: `(client IP, client port, server IP, server port)`. Client port is a random ephemeral port (49152–65535) the OS picks automatically.

### TCP vs QUIC at transport level

| | TCP | QUIC |
|---|---|---|
| Transport | Kernel-managed | UDP + user-space library |
| Connection tracking | OS (new FD per client) | QUIC library (one UDP FD for all) |
| TLS | Separate handshake (+1 RTT) | Built in (same RTT) |
| HOL blocking | Yes (one lost packet stalls all) | No (streams truly independent) |
| Total connection cost | 2 RTTs | 1 RTT (0-RTT if seen before) |
| accept() needed | Yes | No |

### The complete request flow

```
User types: https://example.com/home
       │
       ▼
DNS: example.com → 93.184.216.34          (D01)
       │
       ▼
First visit? → TCP (default)
Repeat visit? → QUIC (cached Alt-Svc)
       │
       ├── TCP path:                        (D01)
       │   3-way handshake (SYN/SYN-ACK/ACK)
       │   OS creates fd47 in server FD table
       │   TLS handshake → ALPN picks h2    (D04)
       │   2 RTTs total
       │
       └── QUIC path:                       (D03)
           QUIC+TLS in 1 RTT
           One UDP socket fd3, QUIC lib tracks connections
       │
       ▼
Server side:                                (D06)
  Parent listening on fd3 (port 443)
  accept(fd3) → fd47 (new connection socket)
  Concurrency model handles fd47:
    Process → fork child
    Thread  → grab from pool
    Async   → add to epoll, create coroutine
       │
       ▼
HTTP version shapes requests:               (D03)
  HTTP/1.1 → sequential on fd47
  HTTP/2   → multiplexed streams on fd47
  HTTP/3   → independent QUIC streams on fd3 (UDP)
       │
       ▼
Server reads request                        (D02)
Queries DB → await → coroutine pauses (async)
           → read() → thread BLOCKED (thread model)
       │
       ▼
Response sent on fd47                       (D05)
HTTP status code, headers, body
REST conventions applied
```

---

## Quiz Questions + Answers

**Q1.** A Python web server uses threads and gets 1,000 concurrent requests. Each request spends 40ms waiting for a DB query. Will threads actually run in parallel on 8 CPU cores? Why or why not?

**A1.** No. Python's GIL means only one thread executes Python bytecode at a time, even on 8 cores. The 1000 threads exist as real OS threads, but they're concurrent (taking turns) not parallel. However, during the 40ms DB wait each thread releases the GIL — so other threads can run their Python code during that window. For I/O-bound work like this, threads are still useful despite the GIL.

---

**Q2.** You have a single-threaded async server (like Node.js). A request comes in that needs to resize a 50MB image — pure CPU work, takes 2 seconds. What happens to all other incoming requests during those 2 seconds?

**A2.** They all freeze. The CPU work never hits an `await`, so the coroutine never yields back to the event loop. The single thread is stuck doing image processing for 2 seconds — no other coroutine runs, no other request is handled. Fix: offload CPU work to a separate process or thread pool, then `await` the result.

---

**Q3.** What is the difference between blocking I/O and non-blocking I/O at the OS level? What syscall does the OS provide to efficiently watch many sockets at once?

**A3.** Blocking I/O: thread calls `read()`, OS puts it to sleep (BLOCKED) until data arrives — thread does nothing. Non-blocking I/O: `read()` returns immediately if no data is ready, thread continues doing other work and checks later. The OS syscall for watching many sockets efficiently is `epoll` (Linux) or `kqueue` (macOS) — one thread registers many FDs with it and `epoll_wait()` blocks until any one of them has data, then returns which ones are ready.
