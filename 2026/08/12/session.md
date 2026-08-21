# Mock Interview — Junior Backend Developer
**Date:** 2026-08-12

**Categories covered (3-question session, seed-random start (seed=1892, start=3, step=3), categories 3/4/6 skipped because they repeat 2026-08-11's topics — Databases, Spring Boot & JPA/Hibernate, Concurrency & Locking — sequence otherwise: System Design & Architecture → OOP/Design Patterns & LLD → REST & API Design; Web Security and Core Java, next in the seeded sequence, were dropped when the session was trimmed from 5 questions to 3):** System Design & Architecture Basics, OOP/Design Patterns & Low Level Design, REST & API Design

---

## Q1 — System Design & Architecture Basics

**Question:** Your team adds a cache in front of the "get product price" endpoint because it's hammered with reads and the DB was struggling. A few days later, support gets a ticket: a customer was charged the old price for an item — marketing had updated the price in the admin panel 8 minutes earlier, but checkout still read the stale value from the cache.

What actually went wrong here, and what would you have needed to think about *before* adding that cache to avoid this specific failure? Would you say "don't cache it" is the right lesson, or is there a better fix?

**Candidate's answer:**
> "first i think the wrong here is the way they invalidate the data they set the TTL is too long for that so the order have the old price than the price appearing in the shop / the first i need to think when add the cache is set the TTL more lower and which query can affect to this data on cache so we can invalidate it properly / i think we should keep the cache approach and we need to improve 2 thing i mentioned above"

**Feedback given:**
- Strong: correctly diagnosed the root cause (too-long TTL) and rejected "don't cache it," instead proposing a genuine two-part fix — shorter TTL and invalidating the cache when the underlying data changes — without needing a hint.
- Missing: shortening TTL only shrinks the risk window, it doesn't close it — there's still a race between the DB write and cache expiry/invalidation.
- Missing: the sharper framing that caching is a staleness-vs-load trade-off, and how much staleness is tolerable depends on the cost of being wrong — stale display price is annoying, stale price that gets charged is a money/legal problem.
- Missing: for checkout specifically, the safer design splits the read — cache the price for *display* with a short TTL + invalidate-on-write, but re-verify the authoritative price from the DB at the moment of actually charging the card, so even a missed invalidation can't become a wrong charge.

**Score: 7/10**

---

## Q2 — OOP, Design Patterns & Low Level Design

**Question:** You're reviewing a teammate's PR for a `UserService` class. It currently: creates/updates users in the DB, sends a "welcome" email on signup, and writes an entry to an analytics table every time a user logs in. Your review comment is: "this violates SRP."

Your teammate replies: "It's all about the `User` object though — feels like one responsibility to me. What's actually wrong with it, and what would break later if I leave it like this?" How do you answer them, and how would you restructure the class?

**Candidate's answer:**
> "it violate that SRP in SOLID because we know this is what need from user but when other thing like send notification it usually send from email so when we here we need to write the send email it is not essetial / so that we should create the EmailService so when the user logic change we need to change code and it will take a long time / so we need to restructure the class we should create the EmailService and the Welcome when entry in other utility class so when we need to use it in anywhere in the project we just inject it and call it"

**Feedback given:**
- Strong: correctly called out the SRP violation and landed on the right refactor shape — extract email-sending and analytics-logging into their own classes and inject them into `UserService`.
- Missing: the precise SRP framing ("a class should have only one reason to change") — right now `UserService` has three reasons to change (user rules, email content/provider, analytics schema), and any of them forces a touch/redeploy of the same class.
- Missing: the concrete "what breaks later" — testing user-creation logic now drags in mocking email/analytics; reusing "welcome email" logic from another path (e.g., admin bulk import) means duplicating it or rerouting through user creation.
- Missing: the Dependency Inversion half of this question — `UserService` should depend on an interface (e.g., `NotificationSender`), not a concrete `EmailService`, so swapping the implementation later doesn't require touching `UserService` at all. "Inject it and call it" gets SRP; depending on an abstraction is what gets DIP.

**Score: 5/10**

---

## Q3 — REST & API Design

**Question:** A partner team integrating with your `/orders` API keeps filing tickets saying "your docs are wrong" — but when you check the Swagger/OpenAPI spec, every field, type, and required/optional flag is actually accurate. What's missing is that there isn't a single example request or response anywhere in the docs, just the schema definitions.

Why would accurate field-level docs still fail an integrating team in practice, and what specifically would you add to fix it?

**Candidate asked for the question to be explained in more detail** (received an expanded scenario with concrete examples: date-format ambiguity, empty string vs. omitted field, nested `items` shape) before answering.

**Candidate's answer:**
> "it still fail i think because we don't add mor specific case in the request body or some thing when they call the endpoint in the API so if have the error we don't log the error obviously so the request will fail / i think we should add to the swagger some problem like which field are optional the in the body like you mentioned so when they call api wrong it should log the error or part they are missing"

**Feedback given:**
- Strong: sensed there's a real gap beyond "the schema is technically right."
- Missing: the scenario stated required/optional flags were already accurate, so re-stating "tell them which fields are optional" doesn't add anything new.
- Missing: error logging happens *after* a bad request is sent — it doesn't stop the partner team from sending the bad request in the first place. The fix is upstream of that.
- Missing: the actual answer — add concrete `example`/`examples` payloads directly in the OpenAPI spec (a plain order, an order with a discount code, a minimal order). A type declaration like `discountCode: string` doesn't tell you whether an empty string behaves differently from an omitted field, what date format is expected, or what a valid nested `items` entry looks like — a worked example answers all of that without a failed request first. This is why "examples matter more than prose" is standard advice for API docs.
- Two rounds of hint/clarification were given and the core concept still wasn't reached.

**Score: 3/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | System Design & Architecture Basics — caching a price lookup, TTL vs. invalidation-on-write, staleness risk for money-relevant data | 7/10 |
| 2 | OOP, Design Patterns & LLD — SRP violation in `UserService` (email + analytics baked in), and dependency inversion via injected interfaces | 5/10 |
| 3 | REST & API Design — why a schema-accurate OpenAPI spec still fails integrators, and why concrete examples beat prose | 3/10 |

**Overall: 5.0/10**

**Strengths:**
- Q1 was the strongest showing: identified the real root cause (TTL) unprompted and reached for a genuine two-part fix (shorter TTL + invalidate-on-write) without needing a hint — the right instinct for a cache-correctness problem.
- Willing to reason out loud and commit to a judgment call ("keep the cache, improve these two things") rather than hedging.

**Areas to review before the real interview:**
- Precision in "why," not just "what": on Q1 and Q2 the fixes proposed were defensible but the justification was vague ("it will take a long time," "not essential") rather than the sharp mechanism (staleness-vs-risk trade-off; "single reason to change"). Practice stating the *why* as precisely as the *what*.
- Dependency Inversion specifically: reaches for "inject it" (that's DI) but hasn't yet connected it to depending on an *interface* rather than a concrete class — that's the actual DIP lesson and it's what lets an implementation be swapped without touching the consuming class.
- API documentation: the concept that a type-accurate schema is not the same as usable docs, and that runtime error handling is not documentation — this needs a re-read, since two hints didn't land it. Look up OpenAPI's `example`/`examples` keyword specifically.
