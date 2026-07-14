# Mock Interview — Junior Backend Developer
**Date:** 2026-07-14

**Categories covered (seed=2135, start=6, step=3):** Concurrency & Locking → System Design & Architecture Basics → OOP, Design Patterns & LLD → REST & API Design → Web Security & Frontend Basics

---

## Q1 — Concurrency & Locking

**Question:** You're reviewing a coworker's PR. It's a small Spring Boot service with an in-memory request counter (`private int count = 0; increment() { count++; }`). Under low traffic the numbers look right, but under concurrent load the final count comes out lower than the number of requests actually handled. What's going wrong, and how would you explain it in the PR comment?

**Answer:**
> it happen wrong because the way thread work if it have more than 1 thread it will cause the problem... [candidate traced increment() as 3 steps: get value, increment, set value; walked through two threads A and B both reading 10, both writing 11, losing an increment; proposed AtomicInteger, synchronized, or locking as fixes]

**Score: 8/10**

**Strong:**
- Correctly identified this as a race condition caused by `count++` not being a single atomic operation — broke it into read → increment → write, the right mental model.
- Interleaving trace (both threads read 10, both write 11, one increment lost) is a textbook correct lost-update explanation.
- Correct fixes named: `AtomicInteger`, `synchronized`, locking.

**Missing:**
- Never named the concept explicitly ("race condition" / "lost update") — worth stating directly in a PR comment.
- Didn't explain *why* it's not atomic at the JVM level (separate bytecode instructions, no ordering guarantee across threads).
- Didn't distinguish that `AtomicInteger` alone suffices here (single variable) vs. `synchronized`/locking being needed once the invariant spans multiple fields.

**Tip:** When you spot `x++` on a field touched by more than one thread, treat it as a reflex trigger to ask "is this atomic?"

---

## Q2 — System Design & Architecture Basics

**Question:** Your team lead says: "We need a simple `Book` management API — add, view, update, delete books, nothing fancy." Walk me through how you'd structure it: what layers exist, what does each own, and roughly what would the schema and endpoints look like?

**Answer:**
> layered architecture with controller/service/repository folders; BookController with POST/GET/PATCH/DELETE on `/api/v1/books`; BookService handles business logic incl. delete-as-status-update; BookRepository talks to DB; schema: Book { UUID id, String name, category, authorName, publisher, Instant createdAt, updatedAt }

**Score: 7/10**

**Strong:**
- Correct layer split with clear ownership (controller/service/repository).
- Sensible REST endpoints with correctly chosen HTTP verbs (POST/PATCH/DELETE).
- Reasonable schema: surrogate UUID key plus audit timestamps.

**Missing:**
- No list/search endpoint (`GET /api/v1/books`) — only single-book fetch by id.
- Mentioned soft-delete-vs-hard-delete but didn't commit to one or justify the choice.
- Used the `Book` class as both DB entity and implicit API request/response body — no DTO separation.

**Tip:** Checklist "list, get-one, create, update, delete" before finalizing any CRUD design.

---

## Q3 — OOP, Design Patterns & LLD

**Question:** A teammate's PR replaced a simple 2-branch `if (customer.isStudent()) ... else ...` discount check with a full `DiscountStrategy` interface, two implementations, and a `DiscountStrategyFactory` — for what's still just that one two-branch check. What would you say in review? Separately, name a design pattern you're probably already using in a Spring Boot app without having thought of it as a "pattern."

**Answer:**
> original if/else was fine for only 2 discount types — Strategy+Factory here is redundant and adds complexity; @RestController/Spring beans relate to Singleton Pattern; interface with multiple implementing classes sharing common fields is Strategy Pattern; also claims to have used Facade Pattern by "having all things into 1 class"

**Score: 6/10**

**Strong:**
- Good judgment on the over-engineering call — recognized the original if/else was appropriately sized and Strategy+Factory added indirection without payoff.
- Correctly connected Spring beans/`@RestController` to the Singleton pattern (container-managed single shared instance).

**Missing:**
- Facade description is incorrect: "all things in 1 class" is closer to a *God class* anti-pattern, the opposite of Facade (which gives a simple unified interface to a *complex multi-class subsystem*, e.g. a `CheckoutFacade` coordinating `InventoryService`/`PaymentGateway`/`NotificationService`).
- Strategy identification ("interface + implementers with a common field") describes polymorphism generally, not Strategy specifically (which is about swapping *behavior/algorithm* at runtime).
- Didn't state what would flip the original judgment (e.g., 5+ discount types, or rules that change per campaign).

**Tip:** Identify a pattern by the problem it solves first ("simplify a complex subsystem" = Facade; "swap an algorithm at runtime" = Strategy), not by its shape.

---

## Q4 — REST & API Design

**Question:** Back to the `Book` API — the missing `GET /api/v1/books` list endpoint gets added with no pagination, and the catalog grows to 2 million rows. What breaks, and how would you fix it? Separately: if a `PUT` request to update a book times out and the client retries, is that safe? What about a `POST`?

**Answer:**
> full memory load / server crash without pagination or streaming; would add pagination/streaming loading ~10-50 records at once; on retries: "it don't save when we retries the update method because it can work while the request time out" (POST not addressed)

**Score: 6/10**

**Strong:**
- Correctly identified the memory/server-crash risk from an unbounded query and proposed pagination as the fix.

**Missing:**
- Idempotency answer was unclear. Correct answer: `PUT` is idempotent — repeating the same update leaves the same final state, so retrying after a timeout is safe. `POST` usually isn't — if the original request actually succeeded before the timeout, retrying risks creating a duplicate resource.
- POST half of the question wasn't addressed at all.
- Didn't mention the fix also belongs at the SQL level (`LIMIT`/`OFFSET` or cursor-based), not just smaller batches in the app.

**Tip:** Idempotency test — "if I fire this exact request twice, is the end state different?" Same → safe to retry. Different → not safe.

---

## Q5 — Web Security & Frontend Basics

**Question:** React frontend on `localhost:3000` calling Spring Boot on `localhost:8080` — a `fetch` call triggers a CORS error in the console. Why does the browser block this, whose job is it to fix, and what does the backend need to send back? Separately, what would you do around that fetch call for loading and error states?

**Answer:**
> CORS happens because frontend/backend have different origins (different ports); browser blocks the frontend from reading the response; fix via `@CrossOrigin` or origin config; backend must send Allow-Origin header; frontend: separate ErrorComponent fed by an error-message prop, and a circular spinner for loading

**Score: 7/10**

**Strong:**
- Correct and complete on the mechanism: different origin, and specifically that the browser blocks *reading the response* rather than blocking the backend from processing the request — a nuance many juniors miss.
- Correct fixes: `@CrossOrigin`/global CORS config, `Access-Control-Allow-Origin` response header.
- Sensible component structure: dedicated `ErrorComponent` fed by a prop, spinner for loading.

**Missing:**
- Stayed at the rendering layer — didn't sketch the underlying state mechanics (`useState` for `loading`/`error`, driven from a `useEffect` with `fetch(...).then().catch().finally()`).
- Didn't mention preflight `OPTIONS` requests (triggered by custom headers/non-simple methods) and the corresponding `Access-Control-Allow-Methods`/`-Headers` — not required for this simple GET, but good to know the trigger condition.

**Tip:** When describing loading/error state, name the `useState` variables first, then the components that consume them.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Concurrency & Locking — race condition on a shared counter | 8/10 |
| 2 | System Design & Architecture Basics — CRUD service design | 7/10 |
| 3 | OOP, Design Patterns & LLD — over-engineering with Strategy/Facade | 6/10 |
| 4 | REST & API Design — pagination and idempotency | 6/10 |
| 5 | Web Security & Frontend Basics — CORS and React fetch states | 7/10 |

**Overall: 6.8/10**

**Strengths:**
- Strong procedural reasoning — consistently walks through *what happens step by step* (thread interleaving, CORS header flow, pagination memory blowup) rather than reciting definitions.
- Good real-world instincts: recognizing when a pattern is overkill for the problem size, catching that CORS blocks reading the response rather than the request itself, defaulting to pagination for large collections.
- Comfortable naming concrete fixes (`AtomicInteger`/`synchronized`, `@CrossOrigin`, pagination) rather than staying abstract.

**Areas to review before the real interview:**
- Idempotency (PUT vs POST) — state directly that PUT is safe to retry, POST risks duplicates.
- Precise pattern definitions — Facade (simplify a complex subsystem) vs. a God class (opposite); Strategy (swap behavior at runtime) vs. generic polymorphism.
- DTOs vs. entities — why not to return a JPA entity directly from a controller.
- React state mechanics — practice narrating the `useState`/`useEffect` pattern behind a data fetch, not just the resulting UI components.
