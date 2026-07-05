# Mock Interview — Junior Backend Developer
**Date:** 2026-07-04

**Categories this session (seed=765, start=6, step=1):** Concurrency & Locking → Testing & Debugging → Web Security & Frontend Basics → System Design & Architecture Basics → Tooling: Git, Build & Environment

---

## Question 1 — Concurrency & Locking (pessimistic locking)

**Q:** You're building a ticket-booking feature. Two requests hit your service at almost the same instant, both trying to book the last remaining seat on a flight. In code review, a teammate suggests: "Let's just do `SELECT * FROM seats WHERE id = ? FOR UPDATE` before we book it." Walk me through: what problem does this actually solve, what's the tradeoff of doing it this way, and when would you reach for this instead of, say, a version-column approach?

**A:** "the problem it solve that it 2 request can not pass simultaneusly it make just 1 request can book the other is not in the same time i don't know the trade off in it"

**Score: 5/10**

**Strong:**
- Correctly identified that FOR UPDATE prevents both requests from proceeding at the same time — the second one has to wait for the first.

**Missing:**
- What's actually happening: `SELECT ... FOR UPDATE` takes a row-level lock in the database. Any other transaction trying to lock that same row has to wait until the first transaction commits or rolls back. That's the "blocking."
- The tradeoff: this is pessimistic locking — you trade throughput for safety. If many requests hit the same row, they queue up and wait, causing slowdowns/timeouts under load. It also introduces deadlock risk if two transactions lock multiple rows in different orders. The lock is only held for the duration of the transaction, so the transaction must be kept short.
- When to use it vs. a version column (optimistic locking): pessimistic locking fits when conflicts are frequent and expensive to lose (e.g. last-seat booking). Optimistic locking (`@Version`, retry on conflict) fits better when conflicts are rare.

**Tip:** Open two DB client sessions, run `BEGIN; SELECT ... FOR UPDATE;` in one without committing, then try the same in the other — watching it hang live makes the concept concrete.

---

## Question 2 — Testing & Debugging (reproducing bugs, debugger vs. logging)

**Q:** A user reports: "Sometimes when I click submit twice fast, my order shows up with the wrong total." You can't reproduce it by clicking normally, and there's nothing obviously wrong in the code when you read it top to bottom. How would you approach tracking this down — what's your actual first move, and would you reach for a debugger, print statements, or something else first? Why?

**A:** "i think first i will debug this code because the debugger can easily reach the bug logic"

**Score: 3/10**

**Strong:**
- Picked a real tool rather than freezing up; it's true a debugger lets you step through logic once inside it.

**Missing:**
- Skipped the first move entirely: reproducing the bug reliably. A debugger is useless until you can trigger the bug on demand. The scenario hints at the repro recipe ("double-click fast") — try to mechanically recreate exactly that condition first.
- Why a debugger is often the wrong first tool for timing-sensitive bugs: pausing execution changes timing, which can make a race condition disappear (a "Heisenbug"). Logging (with timestamps) observes without altering timing and can be added to production.
- Didn't mention checking existing logs or what to log if adding instrumentation (e.g. request ID + timestamp + computed total).
- The scenario smells like a duplicate/concurrent request problem (non-idempotent submit endpoint) — worth naming that hypothesis explicitly.

**Tip:** When a bug report includes a specific trigger ("double-click", "fast", "sometimes"), treat that as the reproduction recipe and recreate it mechanically before reaching for any tool.

---

## Question 3 — Web Security & Frontend Basics (URL-to-page request lifecycle)

**Q:** A teammate asks: "Why does my browser say `net::ERR_CONNECTION_REFUSED` for like half a second before the page loads, and what's actually happening between me hitting Enter on a URL and the page showing up?" Walk me through what happens, at a level you'd explain to that teammate — and where HTTPS fits into that flow.

**A:** "i think the error it occur when the browser reach the website but this have some latency so this error occur in half a second"

**Score: 2/10**

**Strong:**
- Correctly sensed that this is about network communication involving some kind of round trip between browser and server.

**Missing:**
- The actual flow: (1) DNS resolution — domain name → IP address; (2) TCP handshake — SYN/SYN-ACK/ACK before any data is exchanged; (3) TLS handshake (if HTTPS) — encryption negotiation and certificate verification before any HTTP request goes out; (4) HTTP request/response — browser sends request, server responds, browser renders.
- What `ERR_CONNECTION_REFUSED` actually means: not a latency symptom — it means the TCP connection attempt was actively rejected (nothing listening on that IP/port, server down, firewall blocking). A latency/slowness issue would show as a *timeout*, a different failure mode.
- No mention of DNS, TCP, or TLS/HTTPS at all.

**Tip:** Open DevTools → Network tab, click a request, and look at the "Timing" breakdown (DNS lookup, Initial connection, TLS, Waiting/TTFB, Content download) to see the real phases with real numbers.

---

## Question 4 — System Design & Architecture Basics (CRUD service design)

**Q:** You're asked to build a simple Task/Todo API from scratch for a small internal tool. Walk me through how you'd structure it: what layers would you have, what does each own, and roughly what would your database schema look like?

**A:** "i think i will have 3 layer in the layer architecture the controller, where have the CRUD api for each task, and the mark complete api the service, where will handle the bussiness in todo, that we can do anything to do with the logic in this service the last is repository, where is handle the save, get, update the entity with DB and i think the task should have the schema like this UUID id, String taskName, Duration duration"

**Score: 6/10**

**Strong:**
- Solid layered architecture: controller (API surface incl. mark-complete), service (business logic), repository (persistence) — correctly described what each layer owns.
- Reasonable choice of UUID for the ID field.

**Missing:**
- The schema is missing a completed/status field — an inconsistency with the described "mark complete" API, which needs somewhere to persist that state (e.g. `boolean completed`).
- Missing audit fields — at minimum `createdAt` (often `updatedAt` too) for sorting and debugging.
- `Duration duration` is an ambiguous field name — unclear if it means a due date or an estimated time-to-complete. Vague names cause bugs when others use the field differently than intended.
- Didn't map layers to concrete endpoints (e.g. `POST /tasks`, `GET /tasks`, `PATCH /tasks/{id}/complete`, `DELETE /tasks/{id}`).

**Tip:** Before finalizing a schema, walk through each endpoint and ask "what does this operation need to read or write?" — that catches missing fields before writing code.

---

## Question 5 — Tooling: Git, Build & Environment (commit hygiene)

**Q:** You're fixing a small bug. While you're in there, you also reformat some unrelated code and fix a typo in a comment a few functions away. You open a PR with all of it as one commit. Your teammate comments: "Can you split this up before I review it?" What's the reasoning behind that request, and how would you decide what should go into one commit versus separate ones?

**A:** "i don't know" → *(hint given: "think about what happens later when someone needs to git blame a specific line, revert just one change, or review your PR line-by-line without unrelated noise mixed in")* → "i think if don't split it it will hard to review the code"

**Score: 4/10** *(needed one nudge)*

**Strong:**
- Correctly identified that mixing unrelated changes makes review harder.

**Missing:**
- Git blame usefulness: bundling changes means `git blame` on reformatted/typo-fixed lines points to an unrelated commit message. Separate commits keep blame meaningful.
- Revertability: if the bug fix needs reverting later, a bundled commit forces an all-or-nothing revert, losing the harmless changes too.
- Signal-to-noise in the diff: reformatting can touch many lines with zero functional change, burying the lines that actually matter for review.

**Tip:** Adopt "one logical change per commit": before committing, run `git diff` and ask "does every line here belong to the same story?" Use `git add -p` to split hunks if not.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Concurrency & Locking — pessimistic locking (`SELECT ... FOR UPDATE`) | 5/10 |
| 2 | Testing & Debugging — reproducing bugs, debugger vs. logging | 3/10 |
| 3 | Web Security & Frontend Basics — URL-to-page request lifecycle | 2/10 |
| 4 | System Design & Architecture — CRUD service design | 6/10 |
| 5 | Tooling: Git — commit hygiene | 4/10 |

**Overall: 4.0/10**

**Strengths:**
- Clear grasp of layered architecture (controller / service / repository) and what belongs in each layer.
- Correct base intuition on locking: understands that it serializes concurrent access.
- Honest under uncertainty ("I don't know") rather than guessing confidently — good instinct, just needs more depth behind it.

**Areas to review before the real interview:**
- Request lifecycle fundamentals (DNS → TCP handshake → TLS/HTTPS → HTTP request/response) — the weakest answer this session and a very common interview topic.
- Debugging methodology — reproduce reliably before reaching for tools; understand why debuggers can distort timing-sensitive bugs and when logging is the safer first move.
- Locking tradeoffs — go one level deeper than "it blocks the other request": blocking cost under load, deadlock risk, and optimistic vs. pessimistic tradeoffs.
- Schema completeness — cross-check every field against the actual operations/endpoints it needs to support.
- Git commit hygiene reasoning — know the "why" (blame, revert, review signal) not just "it's easier to review."
