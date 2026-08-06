# Mock Interview — Junior Backend Developer
**Date:** 2026-08-06

**Categories covered (3-question session, seed-random start, categories steered away from 2026-08-05's topics — Spring/JPA, Testing & Debugging, Git Tooling — per explicit request):** Core Java & CS Fundamentals, System Design & Architecture Basics, Web Security & Frontend Basics

---

## Q1 — Core Java & CS Fundamentals

**Question:** A teammate's PR has a test where `Integer a = 100; Integer b = 100;` and `a == b` prints `true`, but `Integer c = 200; Integer d = 200;` and `c == d` prints `false`. Same pattern, same operator — why the difference, and what should they use instead?

**Candidate's process:** Asked for a hint, then guessed "it will divide the way it wrap based on the value" (right direction, no mechanism). After a second hint, explicitly said "I don't know."

**Feedback given:**
- Missing: `Integer a = 100` autoboxes via `Integer.valueOf(int)`, not `new Integer(100)`.
- Missing: `Integer.valueOf()` caches instances for values **-128 to 127** (`IntegerCache`) — same object returned every time in that range, so `==` "works" by accident.
- Missing: outside that range (200), `valueOf()` allocates a new object each time — `==` compares references, not values, so it's `false` even though the values are equal.
- Missing: the fix — always use `.equals()` for wrapper-type comparison, never `==`.
- Reflex given: treat `==` next to any wrapper type (`Integer`, `Long`, `Boolean`, `Character`) as a bug until proven otherwise.

**Score: 3/10** — needed two hints and still couldn't reach the mechanism independently.

---

## Q2 — System Design & Architecture Basics

**Question:** An app stored logged-in sessions in an in-memory `Map` on a single server. A second instance was added behind a round-robin load balancer, and users started getting randomly logged out mid-session. What's happening, and what are the fix options?

**Candidate's answer (after one nudge toward "what's different about the request path now"):**
> "if log in request go to the first server and save the Map in the first server... but when have more user the load balancer will route request to server [2] where this map does not save any session of user... so if we want to fix that... we need to use the DB that 2 server can read into it we should use Redis"

**Feedback given:**
- Strong: correctly traced the exact mechanism — session written to one instance's local memory, invisible to the other instance, load-balancer routing makes it look random. Correctly proposed externalizing to Redis. No further hint needed after the nudge.
- Missing: the general principle — **services should be stateless**; anything in a singleton/in-process field breaks the instant there's more than one instance (same shape as an in-memory cache or rate limiter).
- Missing: the alternative fix, **sticky sessions** (route the same client to the same server via cookie/IP hash) — simpler to bolt on, but fragile (session lost if that instance restarts) vs. the more robust externalized-store approach chosen.

**Score: 7/10** — correct core mechanism and fix, one relevant principle/alternative missing.

---

## Q3 — Web Security & Frontend Basics

**Question:** A teammate proposes storing `user=John;role=admin` directly in a cookie so every request can read it without a DB hit. What's wrong with this, and what should they use instead?

**Candidate's answer (after clarifying whether the cookie is visible to others, and being told the user themselves can edit it via DevTools):**
> "i will not agree with my teammate because the data in cookie can be attacked by hacker and they can see some personal information... i think i will point them to use the JWT token because user just log in once and will use this token to attach in the header or its request so that validate it in server side it is safe"

**Feedback given:**
- Strong: disagreed with the insecure proposal immediately; landed on JWT unprompted.
- Missing: mischaracterized the risk as "hacker sees personal info" (confidentiality) when the scenario (and the hint given) pointed at **the user tampering with their own cookie to self-escalate to admin** (integrity) — a different, more direct vulnerability than eavesdropping.
- Missing: *why* JWT actually fixes it — it's **cryptographically signed**, so any edited claim (like `role`) invalidates the signature and the server's verification step rejects it. "Validate server-side" alone isn't the mechanism; signature verification is.
- Missing: the alternative — traditional server-side sessions (cookie holds only an opaque session ID; real data looked up server-side) as a second valid answer to contrast against JWT.

**Score: 6/10** — correct disagreement and correct tool named, but the specific vulnerability and the actual signing mechanism weren't precise.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Core Java — Integer caching / `==` vs `.equals()` on wrapper types | 3/10 |
| 2 | System Design — stateless services / session sharing across instances | 7/10 |
| 3 | Web Security — cookie tampering vs JWT signing | 6/10 |

**Overall: 5.3/10**

**Strengths:**
- Traced the multi-instance session bug to its exact mechanism (in-memory map isolated per server) with only a light nudge, and proposed the standard fix (externalize to Redis) — matches the strongest System Design answer pattern seen in prior sessions.
- Good security instinct: disagreed with an insecure proposal immediately and reached for a real auth mechanism (JWT) without being told to.
- Asks clarifying questions before committing to an answer (session expiry, cookie visibility) rather than freezing or guessing.

**Areas to review before the real interview:**
- **Integer caching / autoboxing** — `Integer.valueOf(int)` caches instances for -128..127, so `==` "accidentally works" for small values and breaks for larger ones; always use `.equals()` for wrapper-type comparison. Needed two hints and still couldn't land it independently — a fresh gap.
- **Precision on "what's actually vulnerable"** — when client-held data is involved, name the specific failure mode (tampering/integrity vs. eavesdropping/confidentiality) rather than defaulting to "a hacker could see it." They're different risks with different fixes.
- **Name the mechanism, not just the tool** — "use JWT" is correct, but the sharper answer is "use JWT *because it's cryptographically signed, so the server can detect tampering*." Same pattern everywhere: Redis fixes session sharing *because* it's externalized, not just because it's Redis.
