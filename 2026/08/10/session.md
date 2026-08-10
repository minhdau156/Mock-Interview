# Mock Interview — Junior Backend Developer
**Date:** 2026-08-10

**Categories covered (3-question session, seed-random start (seed=1618, start=9, step=3), categories 2/3/4 skipped because they repeat 2026-08-09's topics — OOP/LLD, Databases, Spring/JPA — sequence otherwise: System Design → REST & API → Web Security):** System Design & Architecture Basics, REST & API Design, Web Security & Frontend Basics

---

## Q1 — System Design & Architecture Basics

**Question:** Your manager pulls you aside on a Monday: "We're launching a campaign next week and marketing wants to share short links instead of our long article URLs — something like bit.ly, just for us. Can you sketch the design before standup?"

Walk me through:
- What would the database schema look like (the key table(s) and fields)?
- What are the 1–2 core endpoints you'd expose?
- Where do you think this simple design would start to break down as traffic grows — and is there a piece of it you'd deliberately *not* build yet, to avoid over-engineering a small service?

**Candidate's answer:**
> "first we need the table have 3 column is the id, the short url and the real url / the first endpoint is get in this we will call the real endpoint it will call to DB and get the real url and call it / when traffic grow it will double our request to our server"

**Feedback given:**
- Strong: correctly identified the core lookup shape (id, short code, real URL) and correctly identified the GET/redirect endpoint as the read-path core operation.
- Missing: only one endpoint was described — the write path (`POST /shorten` to create a new short code from a long URL) was left out entirely; without it there's no way short URLs ever get created.
- Missing: didn't call out that the short-code column needs a unique index, since it's the lookup key for every redirect.
- Missing: "it will double our request to our server" doesn't name an actual failure point — the real answer is that every redirect is a DB read, so the database becomes the bottleneck first; the standard fix is a cache (e.g. Redis mapping code→URL) or a read replica.
- Missing: didn't address what to deliberately not build yet (e.g. click analytics, multi-region replication, a fancy collision-resistant ID scheme) to avoid over-engineering a v1.

**Score: 4/10**

---

## Q2 — REST & API Design

**Question:** In a PR, you see this endpoint:

```
POST /orders
```

On success it returns `200 OK` with the created order in the response body. If the `customerId` in the request doesn't exist, it returns `500 Internal Server Error`. If the order has no items, it also returns `500 Internal Server Error`.

A reviewer comments: "These status codes are going to cause problems for whoever consumes this API." What's wrong here, and what would you change?

**Candidate's answer:**
> "i think the wrong is the error message for each error is not suitable in it case so we hard to know what is error and we need to check log and if the log don't exist we will not have any evidence in this error / so first we need to create the exception for each case like NoItemException return 409, NoCustomerIdException return 400 / so we should use the Global Handle Error like RestControllerAdvice to listen any the exception throw and handle it"

**Feedback given:**
- Strong: good instinct on diagnosability — `500` for client-caused problems hides the real cause; correctly proposed custom exceptions plus a global handler (`@RestControllerAdvice`) as the Spring pattern for consistent responses; `NoCustomerIdException → 400` is a reasonable call.
- Missing: didn't touch the biggest issue in the snippet — `200 OK` on a `POST` that creates a resource should be `201 Created` (ideally with a `Location` header), which is exactly the kind of inconsistency the reviewer flagged.
- Missing: `NoItemException → 409` isn't right — `409 Conflict` means the request conflicts with the resource's current state; "order has no items" is a validation failure on the request itself, so it belongs with `400`/`422`.
- Missing: didn't mention the response *shape* — a consistent error body (`code`, `message`, `fieldErrors`) matters as much as the status code.

**Score: 5/10**

---

## Q3 — Web Security & Frontend Basics

**Question:** A junior teammate asks you: "Our login page sets something called a session cookie, but the mobile app team says they use a JWT in an `Authorization` header instead. Isn't that just two ways of doing the same thing?"

How would you explain the difference to them — and is there a situation where you'd deliberately pick one over the other?

**Candidate's answer:**
> "no it not doing the same thing / the first is stateful and the second is stateless / so when we use two way to handle the login it will conflict / because the JWT will attach to header and session id is also but when the request go to my app i need to handle both of them, validate the session_id and jwt so it will make the app more complex and hard / in my opinion i will pick JWT for both because the jwt is stateless it don't save any info about the user like using session in statefull, it just need validate the JWT using the secret so when valid it just pass through ortherwise the statefull will make our memory in server is higher"

**Feedback given:**
- Strong: correctly identified the core distinction (stateful session vs. stateless JWT) and reasoned about the memory-cost consequence of holding per-user state server-side.
- Missing: the mechanism isn't quite right — a session ID is normally attached automatically by the browser via the `Cookie` header, while a JWT is attached manually by client code to the `Authorization` header; that automatic-vs-manual distinction is part of why they suit different clients.
- Missing: "it will conflict" overstates it — a backend can run both schemes at once via separate auth filters; the real cost is maintaining two auth paths, not a hard conflict.
- Missing: the main trade-off — revocation. A session can be invalidated instantly server-side; a JWT stays valid until expiry regardless, since the server isn't tracking it (needs a blocklist or short expiry + refresh tokens to match session behavior).
- Missing: storage security — a session cookie can be `httpOnly` (unreadable by injected JS/XSS); a JWT in `localStorage` for header-based sending is directly readable by any injected script.

**Score: 5/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | System Design — URL shortener: schema, endpoints, and where it bottlenecks under load | 4/10 |
| 2 | REST & API Design — status codes for a `POST /orders` endpoint (201 vs 200, 400/422 vs 409) | 5/10 |
| 3 | Web Security — session cookies vs. JWT, stateful vs. stateless, revocation trade-off | 5/10 |

**Overall: 4.7/10**

**Strengths:**
- Consistently reaches for the right *category* of fix on the first try — constructor injection (prior session), custom exceptions + global handler, stateless vs. stateful — showing the fundamentals are there.
- Reasons about consequences rather than reciting rules (memory cost of session state, log-diagnosability of vague error codes).

**Areas to review before the real interview:**
- REST design: for `POST` endpoints that create a resource, default to `201 Created`; reserve `409 Conflict` for state conflicts, not plain validation failures (use `400`/`422` instead).
- System design answers: always name a *specific* resource that gets hot under load (DB reads, a lock, a single instance) instead of a general "it gets slower" statement, and explicitly call out what you'd deliberately defer to avoid over-engineering.
- Auth: JWT vs. session isn't just about memory — revocability and where the token can be stolen from (cookie + `httpOnly` vs. header + `localStorage`) usually decide it in practice.
