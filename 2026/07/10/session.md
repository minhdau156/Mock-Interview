# Mock Interview — Junior Backend Developer
**Date:** 2026-07-10

## Categories this session
Seed = 10×137 + 7×31 = 1587 → starting_index = 8, step = 7
1. Web Security & Frontend Basics
2. REST & API Design
3. OOP, Design Patterns & LLD
4. System Design & Architecture Basics
5. Concurrency & Locking

---

## Q1 — Web Security & Frontend Basics

**Question:** You're building a React app that calls your Spring Boot API from `http://localhost:3000`, while the API runs on `http://localhost:8080`. You wire up a `fetch()` call to `GET /api/orders`, and in the browser console you see a request that fails before your controller even logs anything — the network tab shows the browser sent an `OPTIONS` request first, and it got blocked. Walk me through: what is the browser actually doing here, why does it block the real request, and what does your Spring Boot side need to do to let it through? Bonus: what's one CORS config mistake you'd want to avoid in production (versus what's fine for local dev)?

**Candidate answer:** "i think the CORS is send the type OPTIONS request first and then so if we don't have the config allowed the OPTIONS it will be blocked"

**Score: 5/10**

**Strong:**
- You correctly identified that this is a CORS preflight — the browser sends an `OPTIONS` request before the real one, and if the server isn't configured to allow it, it gets blocked.

**Missing:**
- The "why": this happens because `localhost:3000` and `localhost:8080` are different **origins** (different port = different origin, even on the same host). The browser's **same-origin policy** blocks the frontend JS from reading responses from a different origin unless the server explicitly opts in via CORS headers. It's not the request itself that's blocked (the server does receive it) — it's the browser refusing to expose the response to your JS if the origin isn't allowed. The preflight `OPTIONS` is only sent for "non-simple" requests (JSON body, custom headers, methods like PUT/DELETE) — nuance not covered.
- The fix on the Spring Boot side: `@CrossOrigin` on a controller, or a global `CorsConfigurationSource` bean / `WebMvcConfigurer.addCorsMappings()`, specifying `allowedOrigins`, `allowedMethods`, `allowedHeaders`.
- Bonus not addressed: a common prod mistake is `allowedOrigins("*")` combined with `allowCredentials(true)` (browsers reject this combo outright). In prod, use an explicit whitelist of real frontend domains instead of `*`.

**Tip:**
- When you hit a CORS error, ask "is this a browser-enforced restriction, or did the request actually fail server-side?" — that distinction is the core concept most juniors miss.

---

## Q2 — REST & API Design

**Question:** You're designing an endpoint for canceling an order: `POST /api/orders/{id}/cancel`. A teammate in code review comments: "Why isn't this just `PUT /api/orders/{id}` with a status field?" How do you respond — what's your reasoning for picking one over the other, and does your answer change if you consider what happens when the same cancel request gets sent twice (e.g. the client retries after a timeout)?

**Candidate answer:** "the first i will keep my option because if we cancel it does not update the field , it will delete the process, and or rollback the paid money if they pay before / so if i choose my option i think both of them will cause some bug when the request make twice so that i will add the Idempotency key for the request like this"

**Score: 7/10**

**Strong:**
- Good instinct on the design question: "cancel" isn't just a field mutation — it can trigger side effects (rolling back a payment, releasing inventory), so an explicit action endpoint (`POST /cancel`) makes that intent clearer than hiding it inside a generic `PUT`.
- Correctly spotted the double-submission risk and reached for **idempotency keys** — genuinely solid, above-junior-baseline knowledge (the standard pattern used by Stripe and other payment APIs for exactly this problem).

**Missing:**
- Framed the retry issue as "both will cause some bug" rather than naming it precisely as an **idempotency** problem. Also, a plain `PUT` can actually be idempotent by HTTP semantics (setting `status = CANCELLED` twice yields the same end state) — the real danger is if the cancel logic re-triggers the refund every time it runs regardless of current status. The cheapest first fix is often simpler than a full idempotency-key system: check current state before applying the side effect (`if (order.status == CANCELLED) return early`). Idempotency keys are the heavier tool for cases where state alone can't tell you "have I already processed this exact request."
- Didn't acknowledge the counterargument (PUT's idempotent-by-contract nature is exactly why some teams prefer folding transitions into it) — would show a more rounded answer.

**Tip:**
- Use the word "idempotent" explicitly when discussing retries — and lead with the lighter state-check guard before reaching for the heavier idempotency-key machinery.

---

## Q3 — OOP, Design Patterns & LLD

**Question:** You're reviewing a teammate's `NotificationService`:
```java
public class NotificationService {
    public void notify(String type, String message, String recipient) {
        if (type.equals("EMAIL")) {
            // send email
        } else if (type.equals("SMS")) {
            // send SMS
        } else if (type.equals("PUSH")) {
            // send push notification
        }
        // ... teammate says "we're adding Slack and WhatsApp next sprint"
    }
}
```
What's your concern with this code as written, what pattern would you reach for to clean it up, and how would you sketch the refactor (classes/interfaces, not full code)? Also — is there a point where this refactor would be overkill, e.g. if they said "we'll never add more than these 3 types"?

**Candidate answer:** "i think it will violate the O priciple in SOLID, because if we need change we need to add more the if else so it can cause some bug, / i will use the factory pattern for this each will go to factory pattern first , but it should use in the case have many type in the question if it less than 3, we can use the if else so it don't make the code more complicate and redundant"

**Score: 6/10**

**Strong:**
- Correctly named the violated principle — Open/Closed Principle.
- Good judgment on the "is this overkill?" question — recognizing that with only 2-3 fixed types and no stated growth, a plain if/else is simpler than introducing an abstraction layer.

**Missing:**
- The sketch was thin. A concrete answer: an interface `NotificationSender` with `send(String message, String recipient)`; concrete classes `EmailSender`, `SmsSender`, `PushSender` (later `SlackSender`, `WhatsAppSender`) implementing it; a `NotificationSenderFactory` (or `Map<String, NotificationSender>` injected by Spring) that returns the right implementation for a given `type`. `notify()` then becomes `factory.get(type).send(message, recipient)` — no if/else at all.
- "Factory pattern" alone is the *selection* mechanism, but doesn't by itself remove the if/else — it's the combination of a shared interface (so implementations are polymorphic) plus the factory/map lookup that does. Worth having both halves in the answer.

**Tip:**
- When naming a pattern, name the actual objects it produces — interface, implementations, and who selects one — not just the pattern label.

---

## Q4 — System Design & Architecture Basics

**Question:** You're asked to design a simple URL shortener (like bit.ly) as a take-home-style exercise — just the backend. Walk me through: what's the core table/schema you'd start with, what does the "create short URL" endpoint do, what does the "redirect" endpoint do, and where do you think the first bottleneck would show up as traffic grows?

**Candidate answer (after 2 rounds of hints):** "i think i will create the table that have 2 column the first column is the short code, i the second is the real url we need to attach to our domain / the create endpoint it will get the url and return the short code that made from this url / i think the redirect endpoint will redirect us to the real endpoint to handle" — then on the bottleneck question: "it will get more traffic because 1 request need more than two call to endpoint so it wiil down our server"

**Score: 4/10**

**Strong:**
- With guidance, landed on the right core schema — a table mapping short code → long URL — and correctly described the create endpoint's contract.
- Right general shape that redirect sends the user to the real URL, even without the mechanics.

**Missing:**
- Needed significant hinting (two rounds) to reach even a basic shape — worth noting for the real interview, where hints are much more limited.
- Redirect endpoint mechanics: an HTTP redirect status code (301 permanent or 302 temporary) with a `Location` header set to the long URL.
- No mention of indexing: `short_code` is looked up on every redirect, so it needs a unique index (often the primary key itself) — otherwise every redirect becomes a slow table scan as the table grows.
- The bottleneck answer didn't land — "1 request needs more than two calls to endpoint" isn't a real mechanism here. The actual answer: a URL shortener is overwhelmingly read-heavy (far more clicks than creations, often 100:1+), so the bottleneck shows up on the redirect/lookup path — high-frequency reads against the DB, typically solved with a cache (e.g. Redis) in front of the DB for hot short codes.

**Tip:**
- For "where's the bottleneck" questions, start by asking "which operation happens way more often, reads or writes?" — that ratio almost always points to the answer.

---

## Q5 — Concurrency & Locking

**Question:** Two support agents open the same customer's account in your admin panel — Agent A updates the support ticket status, Agent B updates the phone number. Both load the record at roughly the same time, both edit different fields, and both click "Save" about 5 seconds apart. When Agent B saves, Agent A's status change is gone — completely overwritten. Walk me through: what's actually happening under the hood with a plain `UPDATE` query that causes this, and what's one concrete way you'd change the code to prevent it?

**Candidate answer (after 1 round of clarification):** "no it don't lnow because it work in 2 different thread so when the agent B update the A don;t know and otherwise, i think we should add the the @Transactional to this because it will have"

**Score: 3/10**

**Strong:**
- Right general instinct that the two requests don't know about each other — each operates on its own snapshot.

**Missing:**
- This isn't really a "different threads don't know about each other" visibility problem (that framing fits shared in-memory state races). Here it's two separate database transactions, each doing a full read-then-write of the row, with no check for "has this row changed since I loaded it?" A plain `UPDATE customers SET status=?, phone=?, ... WHERE id=?` blindly overwrites every column with whatever's in memory.
- Important misconception: `@Transactional` does not fix this. It wraps a single method in one atomic transaction — it does nothing to coordinate between two separate transactions from two separate requests. Agent A's save and Agent B's save already each run in their own transaction; adding `@Transactional` again changes nothing.
- The two real fixes: (1) **Optimistic locking** — a `@Version` column on the entity; Hibernate issues `UPDATE ... WHERE id=? AND version=?`, and if another transaction already bumped the version, 0 rows match and Hibernate throws `OptimisticLockException` (catch it, retry or report). (2) **Pessimistic locking** — `SELECT ... FOR UPDATE` when loading, blocking other transactions until yours commits — better for short, high-conflict operations; worse here where conflicts are rare and blocking would hurt responsiveness for no reason.

**Tip:**
- Ask first: "are these racing inside the same process (needs `synchronized`/atomics) or racing across time via the database (needs optimistic/pessimistic locking)?" Mixing these up (reaching for `@Transactional`) is a common and costly junior mistake to correct now.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Web Security & Frontend Basics — CORS preflight and same-origin policy | 5/10 |
| 2 | REST & API Design — action endpoint vs. generic PUT, idempotency | 7/10 |
| 3 | OOP, Design Patterns & LLD — OCP violation, Strategy/Factory refactor | 6/10 |
| 4 | System Design & Architecture Basics — URL shortener schema and bottleneck | 4/10 |
| 5 | Concurrency & Locking — lost-update problem, optimistic vs. pessimistic locking | 3/10 |

**Overall: 5.0/10**

**Strengths:**
- Best answer of the session (Q2): correctly reasoned through why an action endpoint beats a generic field update when there are real side effects (refund/rollback), and independently reached for idempotency keys to handle retries — above-baseline junior knowledge.
- Correctly named the Open/Closed Principle violation in the notification service and gave sound judgment on when a pattern refactor is and isn't worth it (Q3).
- Good interview habits: asked for clarification instead of guessing blindly, and reasoned out loud even when unsure (Q4, Q5).

**Areas to review before the real interview:**
- Concurrency vs. transactions (Q5, lowest score): `@Transactional` does not prevent lost updates across separate requests. Study optimistic locking (`@Version` in JPA) and pessimistic locking (`SELECT ... FOR UPDATE`), and practice telling apart "racing in-process" vs. "racing across transactions over time."
- CORS fundamentals (Q1): review why the browser blocks the response (same-origin policy), not just that a preflight `OPTIONS` request exists — and know the concrete Spring-side fix (`@CrossOrigin` / `CorsConfigurationSource`).
- System design bottleneck reasoning (Q4): practice asking "which operation happens far more often, reads or writes?" to locate bottlenecks — a hot key-lookup path (like URL redirects) is a textbook caching case.
- Concrete pattern sketching (Q3): when naming a design pattern, practice naming the actual interface and implementing classes, not just the pattern label.
