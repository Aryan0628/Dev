# Aryan's Learning Personalization

> Read this before every session. It captures how Aryan learns, what to anticipate, and how to craft better responses over time.

---

## How Aryan Learns

- **Catches imprecisions fast.** If something is slightly wrong or loosely stated, he will question it. Example: caught that HTTP/2 "streams" are logical labels on one TCP pipe, not real independent streams like in HTTP/3. Don't be loose with terminology.
- **Asks very foundational follow-up questions.** Not because he's slow — because he's filling gaps in his mental model properly. Example: "so HTTP/3 doesn't use a socket?" is a good question, not a dumb one. Answer it completely, never brush it off.
- **Notices when a word is used at two different layers.** Example: "stream" means something different in TCP, HTTP/2, and HTTP/3. If you reuse a term across layers, he will get confused. Define the layer explicitly every time.
- **Wants acronyms spelled out on first use, every session.** He explicitly asked for this. Example: RTT = Round Trip Time, HOL = Head-of-Line, ALPN = Application-Layer Protocol Negotiation. Never drop an acronym cold.
- **Loves internals.** His questions often go: concept → "but how does X actually work under the hood?" Preemptively answer the internal mechanic, don't wait for him to ask.
- **Prefers short comparison tables** for side-by-side differences.
- **Wants one-line summaries** at the end of complex explanations. He asked for this in D03.
- **Asks about connections between new concept and old ones.** Always link back to what was already covered (D01, D02, etc.).

---

## What to Anticipate Per Topic

When teaching a concept, ask yourself:
1. **Is any word reused with different meaning at different layers?** → Define each separately.
2. **Does something happen on both client AND server side?** → Show both sides explicitly.
3. **Is there an OS vs app distinction?** → Always call out who does what.
4. **Does this rely on a term from a previous session?** → Briefly re-anchor it.
5. **Is there a "but wait, does X actually mean Y?" trap?** → Answer it before he asks.

---

## Response Crafting Rules

- **Define acronyms inline, first use, every session.** Even if covered before — new context window, new session.
- **When showing a flow, show BOTH sides** (client and server) with a diagram.
- **When OS and app are involved, show the boundary explicitly** — what the kernel does vs what the app does.
- **Never use a word at two layers without flagging it.** "Stream" in TCP ≠ "stream" in HTTP/2 ≠ "stream" in HTTP/3. Label them.
- **Short table for comparisons.** Column headers: the things being compared. Rows: the dimensions.
- **End complex explanations with a one-line summary** ("One-line answer: ...").
- **Don't rush past foundational questions.** If he asks "what is a socket," treat it as a full question worth a full answer.
- **Cheatsheet (sessions/Dxx.md) should reflect ALL the follow-up Q&A from that session**, not just the initial lesson structure. Update it after the discussion deepens.

---

## Per-Session Research & Content Standard

### Research approach
- **Primary sources: your own knowledge + web search.** These are deeper and more current than any reference tool.
- **Archon is a supplementary lookup** — glance at it for structure or terminology checks, but never treat it as authoritative. If Archon contradicts what you know, trust what you know.
- **Do not copy-paste from Archon** — synthesise everything in your own words, personalised to Aryan's level.

### Real-world incidents are mandatory
Every session must include **at least one real incident** — a known production failure, outage, or costly mistake related to the topic. Format it like this:

> **Real case: [company/system] — [year if known]**
> What happened: ...
> Root cause: ...
> Cost / impact: ...
> What it would have taken to prevent it: ...

Examples of the kind of cases to include: Knight Capital ($440M in 45 min from a deploy bug), Cloudflare global outage from a BGP misconfiguration, GitHub's MySQL replication lag, Slack's WebSocket thundering herd, etc.

### Priority labelling
Every major concept in the session must be tagged with priority so Aryan knows what to focus on for real systems:

- **[CRITICAL]** — you will encounter this in production; getting it wrong causes outages
- **[IMPORTANT]** — frequently appears in design interviews and real architectures
- **[GOOD TO KNOW]** — useful context, not commonly tested or operationally urgent

### Depth vs breadth
- Go deep on **how it actually works** — not just what it is
- Explain the failure modes — what breaks, how it breaks, why it's subtle
- Call out the **most common mistake engineers make** with this concept (1–2 bullets)
- Do not pad with edge cases that almost never happen — Aryan is learning for real systems, not trivia

---

## Observed Doubting Pattern (update this over time)

| Session | What he questioned | What it revealed |
|---|---|---|
| D03 | "HPACK is shared authentication bearer?" | Misread "shared table" as auth-related — needs precise language |
| D03 | "HTTP/2 has single stream, HTTP/3 has separate — you described wrong?" | Caught logical/real stream distinction — good systems thinker |
| D03 | "HTTP/3 doesn't use a socket?" | Gap: thought socket = TCP specifically |
| D03 | "Handler routing?" | Wanted to connect new transport knowledge back to D02 routing |
| D03 | "When client sends SYN, how does server know to open a socket?" | Gap: didn't know listening socket pre-exists before any client connects |
| D04 | "How does server verify it is trusted client?" | Flipped the model — assumed TLS is bidirectional by default |
| D04 | "Are the public keys the same?" | Confused certificate public key [identity] with ECDHE key share [key exchange] — needs explicit separation of purposes |
| D04 | "Where is private key used?" | Good systems thinking — traced the key's role through the whole flow |
| D04 | "You said only public key is sent but then mentioned private key?" | Caught imprecision — needed clarity that signature travels, not the key |
| D04 | "Why ECDHE if DigiCert already verified?" | Excellent conceptual question — needed explicit separation of identity vs encryption |
| D04 | "ECDHE per packet or per connection?" | Connected new concept to prior TCP/connection knowledge from D01/D03 |
| D04 | "Same client requesting two routes — what happens?" | Connected to D03 multiplexing knowledge — good cross-session thinking |
| D05 | "So REST API is just how to use HTTP? That's all?" | Excellent compression — he distilled the whole session into one accurate insight; confirms he builds correct mental models quickly |
| D06 | "That's why Python is called single-threaded?" | Confused GIL with single-threaded — needed explicit distinction: Python has real threads, Node.js is actually single-threaded |
| D06 | "How can one thread see other sockets if each thread has its own stack?" | Gap: thought sockets were thread-owned — needed explicit clarification that sockets are process-level FDs, any thread can access any FD |
| D06 | "Does a process need threads to exist?" | Excellent foundational question — wanted to confirm the process/thread relationship; every process needs at least 1 main thread |
| D06 | "So when a thread sleeps timing does not apply, when epoll returns then timing applies?" | Perfect synthesis — confirmed he correctly understood BLOCKED state and when timer re-applies |
| D06 | "What is the difference between thread blocking and async event loop?" | Connected the similarity (both BLOCKED, timer off) but wanted the core difference — thread blocked on one socket vs event loop blocked watching all sockets |
| D06 | "If client gets Alt-Svc during TLS does it destroy TCP and switch to QUIC?" | Good sequence thinking — needed to clarify Alt-Svc arrives in HTTP response (after TCP+TLS), not during TLS; no mid-session switching |
| D07 | "Nginx os and my os having node js is different?" | Gap: thought Nginx and Node.js were on different machines/OS — same machine, same OS, two separate processes on localhost |
| D07 | "One process only have one port — but in Model 1/2/3 multiple sockets were made on single port?" | Confused port binding with connections — bind() is done once; many connections can exist on same port via 4-tuple distinction |
| D07 | "Nginx is listening socket so this process forks multiple processes as per request?" | Confused Nginx with Model 1 — Nginx forks workers ONCE at startup (fixed), never per connection; connections go to epoll, not new processes |
| D07 | "In Model 3 also async one client was handled by one process, here multiple clients?" | Slight misremember — Model 3 already handled multiple clients on one thread; Nginx just runs N copies across CPU cores |
| D07 | "So both Nginx and Node.js use async and epoll?" | Great synthesis — confirmed both use Model 3; different FDs, different purposes, same mechanism |
| D07 | "So there are also 1000 connection sockets in Nginx model?" | Good confirmation — yes, 1000 Connection A sockets; but Connection B is a small shared pool (~32), not 1000 |
| D07 | "For Nginx server the req is not client-based exactly — can reuse the connection?" | Spotted the pool insight — Connection B is shared/reused across all clients, not reserved per client |
| D07 | "Client A used connection — next time client B can also use it?" | Confirmed pool is shared across ALL clients — safe because Connection B carries no client identity |
| D07 | "What is Model 3's disadvantage — the 1000 sockets?" | Gap: thought sockets were the problem — every model opens one socket per client; disadvantage is CPU-heavy work on single thread |
| D09 | "Did not understand the state/CSRF check in OAuth" | Abstract description ("trick your server") didn't land — needed a concrete attacker/victim story with named roles and step-by-step sequence to make the attack real |
| D09 | "Wanted the full OAuth flow shown with browser + server + storage simultaneously" | Prefers parallel column format when multiple components interact at the same step — show what each layer sees at every moment, not just a single-thread narrative |
| D14 | "ECDHE will be used to decrypt HTTP frames right?" | Gap: confused ECDHE (key exchange mechanism) with AES (actual cipher). Need to make the ECDHE → KDF → AES chain explicit; ECDHE produces the secret, AES does the encrypting |
| D14 | "Tell the difference between hashing, encryption, and signature" | Gap: three concepts blurred together. Needed a side-by-side table with reversibility + key + purpose + example, then how all three appear in TLS specifically |
| D14 | "What is transcript and all here?" | Abstract term "transcript" didn't land — needed a concrete numbered list of actual TLS messages (ClientHello, ServerHello, etc.) plus a diagram showing both sides building it independently |
| D14 | "ECDHE does get exchanged — how it is transmitted in transcript, everything hashed so client cannot get it back?" | Good critical thinking — caught that ECDHE public parts ARE in the transcript (transmitted in plaintext). Needed explicit public/private split: public_A and public_B transmitted fine, secret_A and secret_B never leave each side |
| D15 | "I queried for 20 — it's an internal node — does it store a pointer?" | Caught that my simplified B-Tree diagram was inaccurate. PostgreSQL uses B+ Tree: internal nodes are routing-only; every queryable key with its ctid lives in a leaf node. 20 exists in both internal (separator) and leaf (with ctid). |
| D15 | "What does INCLUDE do in the tree?" | Good internals question — wanted to know the physical placement. INCLUDE columns are stored only in leaf nodes, invisible to internal nodes. Enables Index Only Scan. |

---

## Cheatsheet Standard

Each `sessions/Dxx-topic.md` must include everything from the session — not just the initial lesson but all follow-up questions and clarifications. If a concept was clarified mid-session, it belongs in the cheatsheet. Purpose: if Aryan reads the cheatsheet months later, he should get the full picture including the nuances he discovered through questions.
