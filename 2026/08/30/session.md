# Mock Interview — Junior Backend Developer
**Date:** 2026-08-30

**Categories covered (3-question session, date-based seed (no timestamp available): seed = 30×137 + 8×31 = 4358, start=9, step=9; sequence: System Design & Architecture Basics → Web Security & Frontend Basics → Testing & Debugging; opening category (9) differs from 08/25's opener (4) per the no-repeat-opener rule; all three scenarios were checked against every prior session back to 07/04 to avoid reusing a scenario/mechanism already asked — System Design landed on the untested "design a simple CRUD service end-to-end" sub-topic (scaling, caching, and sync-vs-async in this category were all already covered in prior sessions), Web Security landed on the untested "what happens when you type a URL / HTTPS" sub-topic via a mixed-content angle (CORS and cookie-vs-JWT were already covered three times combined), Testing & Debugging reused "reading a stack trace" but via a genuinely different mechanism than 08-19's single-frame NPE — a wrapped `Caused by:` chain, not tested before):** System Design & Architecture Basics, Web Security & Frontend Basics, Testing & Debugging

---

## Q1 — System Design & Architecture Basics

**Question:** Your manager pulls you aside: "Support agents need a way to leave private notes on a customer account — just a text note with a timestamp, nothing fancy. Support only, customers should never see these. Can you build the API?"

Walk me through: what does the database table look like, what endpoints do you expose, and where does the "only support agents can see this" rule actually get enforced?

**Candidate's answer** (asked for two hints; first pointed at sketching the schema for one note plus separately thinking about whether a UI-only hide would stop anyone, second pointed at what the server already knows about the requester's identity/role on each request):
> "i think we should have the table have the id, content, author, customer, created_at, updated_at
>
> and i think we should have some endpi=oint like Get /customers/{id}/notes
> and Get /customers/notes to get all notes
>
> and i think we should authorize in both backend , frontend
> in the backend we should authorize base on the role in each user having in the jwt token or session, in the frontend we should authorize who can see the note"

**Feedback given:**
- Strong: reasonable schema (id, content, author, customer FK, created_at, updated_at) — including `author` shows good instinct beyond the bare minimum asked.
- Strong: correctly named the backend enforcement mechanism — role carried in JWT/session, checked server-side.
- Missing: no endpoint to actually create a note — the whole feature is "leave a note," and only read endpoints were sketched (`POST /customers/{id}/notes` is missing entirely).
- Missing: `GET /customers/notes` is unscoped, unclear in purpose, and would collide with `GET /customers/{id}/notes` at the routing level — scope creep with no stated use case.
- Missing/misconception: "authorize in both backend, frontend" — the frontend can only decide whether to *display* a note, which is UX, not authorization; nothing stops a direct API call bypassing the frontend. This is the same principle behind hiding a button to "secure" an action, now recurring in a third context.
- Missing: didn't name which layer the backend check lives in (e.g. `@PreAuthorize("hasRole('SUPPORT')")`).

**Score: 4/10**

---

## Q2 — Web Security & Frontend Basics

**Question:** A teammate says: "Our marketing page loads fine over `https://`, but the 'live chat' widget we embedded shows as broken in the browser, and the console has some error about 'insecure content' being blocked. It works fine on plain `http://`."

Walk me through what's actually happening the moment a browser loads a URL — and where in that process this specific chat-widget failure comes from.

**Candidate's answer** (asked for three hints — first that the page's own connection already succeeds so the issue is about a second resource loaded inside it, second a factual clarification of what a chat widget is, third a same-domain-different-protocol thought experiment to separate origin from protocol — then asked for the answer directly):
> "when the browser load the url / it will go to the DBS and extract to know the actual ip and so it will establish the connection like TCP or UDP and it will send to the server have the ip and return again / in my opinion it can not right but i think the location where process the specific is in the establish the connection i think that because have the TLS to secure"
>
> "i think beacuse it in the diferrent origin"
>
> "can you answer it"

**Feedback given:**
- Strong: correctly sketched the general request lifecycle unprompted — DNS resolution → connection establishment → request/response, with TLS gestured at as relevant.
- Missing: the core mechanism — mixed content blocking, triggered by a protocol downgrade (`https://` page loading an `http://` sub-resource) — was not reached independently after three nudges.
- Missing/misconception: attributed the block to differing *origin* (the CORS concept) rather than differing *protocol* — mixed content, CORS, and same-origin policy are three distinct browser security mechanisms triggered by three different things.
- Missing: didn't reach the "why" — an unencrypted sub-resource is a man-in-the-middle injection point that would compromise the whole page's security guarantee.
- Asked for the answer outright rather than testing the same-domain-different-protocol thought experiment offered.

**Score: 3/10**

---

## Q3 — Testing & Debugging

**Question:** A teammate pings you about a production error:

```
com.acme.orders.OrderProcessingException: Failed to process order 4821
	at com.acme.orders.OrderService.process(OrderService.java:58)
	at com.acme.orders.OrderController.checkout(OrderController.java:33)
	at org.springframework.web.method.support.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:150)
	... 40 more lines of Spring internals ...
Caused by: java.sql.SQLException: Connection is closed
	at com.acme.db.ConnectionPool.borrow(ConnectionPool.java:102)
	at com.acme.orders.OrderRepository.save(OrderRepository.java:44)
	at com.acme.orders.OrderService.process(OrderService.java:55)
	... 3 more
```

Where do you actually start reading this, and why isn't `OrderProcessingException` itself the bug?

**Candidate's answer:**
> "i will started read the root cause in this because i can know what the problem that we need to know
>
> like in this the Connection is closed , the connection between the app and DB is closed so it can not make any query to the db and the database throw the error
>
> so i think we should open more connection and use the transactional in this method"

**Feedback given:**
- Strong: correctly identified, without any hint, that you start at the root cause (`Caused by:`) rather than the top exception — the first debugging question this session solved without scaffolding.
- Strong: correctly read `Connection is closed` as meaning the app couldn't reach the DB to run the query.
- Missing: didn't explicitly state why `OrderProcessingException` isn't the bug — it's a deliberate wrapper the service throws around the low-level `SQLException`, which is why the real cause only ever shows up in the `Caused by:` chain.
- Missing/misconception: the proposed fixes don't match this specific error. "Connection is closed" from a pool (`ConnectionPool.borrow`) is a stale/dead-connection symptom, not pool exhaustion (which throws a timeout error instead) — the real fixes are connection validation on borrow or shorter pool idle/lifetime settings, not "open more connections."
- Missing/misconception: `@Transactional` manages commit/rollback boundaries, not connection health — reused a tool that fit an earlier, different problem (JPA session lifecycle) without checking it addresses this failure.

**Score: 5/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | System Design & Architecture Basics — designing a support-notes CRUD API end-to-end: schema, endpoints, and where "support-only" access is actually enforced | 4/10 |
| 2 | Web Security & Frontend Basics — mixed content blocked (`https://` page loading an `http://` chat widget), DNS/TCP/TLS request lifecycle | 3/10 |
| 3 | Testing & Debugging — reading a wrapped exception's `Caused by:` chain to find the real root cause | 5/10 |

**Overall: 4.0/10**

**Strengths:**
- Found the right starting point independently in two different questions this session — reading root cause first in a `Caused by:` chain (Q3, no hint needed), and correctly naming JWT/session role as the backend enforcement mechanism (Q1).
- Reasons out loud and revises mid-answer rather than freezing — walked through the DNS→TCP→TLS flow before landing on TLS as "probably" relevant, the right instinct even when the specific conclusion was off.

**Areas to review before the real interview:**
- **"Authorize on frontend and backend" is a recurring misconception, now seen in a third distinct context** (this session's support-notes API, plus prior sessions' hidden discount button and client-side price validation). Internalize: a frontend check is UX, never a security boundary — test it by asking "if I deleted the frontend check, is the system still secure?"
- **Mixed content vs. CORS vs. same-origin policy are three different browser security mechanisms**, triggered by three different things (protocol downgrade, response-reading permission, and domain/port/protocol together respectively) — don't default to "different origin" whenever a browser blocks something.
- **Connection pool failures aren't all the same failure.** "Connection is closed" (stale/dead connection handed out) and "timeout waiting for a connection" (pool exhaustion) are different root causes with different fixes — and `@Transactional` addresses neither; it only manages commit/rollback boundaries.
- **Commit to a full guess before asking for the answer outright** — Q2 needed three nudges before an explicit "can you answer it," compared to Q3's independent, hint-free read that landed on exactly the right starting point. The gap between those two answers in the same session shows the skill is there; it needs to be trusted under uncertainty instead of deferred at the first sign of doubt.
