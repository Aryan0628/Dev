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

---

## Cheatsheet Standard

Each `sessions/Dxx-topic.md` must include everything from the session — not just the initial lesson but all follow-up questions and clarifications. If a concept was clarified mid-session, it belongs in the cheatsheet. Purpose: if Aryan reads the cheatsheet months later, he should get the full picture including the nuances he discovered through questions.
