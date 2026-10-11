# Mock Interview — Junior Backend Developer
**Date:** 2026-09-06

**Categories covered (3-question session, date-based seed (no timestamp available): seed = 6×137 + 9×31 = 1101, start=2, step=7; sequence: OOP, Design Patterns & Low Level Design → System Design & Architecture Basics → Concurrency & Locking; opening category (2) differs from 09-02's opener (4) per the no-repeat-opener rule; all three sub-topics were checked against every prior session back to 07-04 to avoid reusing a scenario/mechanism already asked — OOP landed on the untested Observer/`ApplicationEvent` angle (Strategy, SRP, vending-machine LLD, singleton mutable state, and DIP-via-hardcoded-`new` had all already been covered), System Design used the untested TODO-API variant of the "junior-scale design question" bullet (URL shortener had been used twice), and Concurrency used the untested JPA optimistic-lock exception-handling angle — retry vs. surface-to-user — distinct from the previously-covered pessimistic-locking flash-sale question, the cross-instance `synchronized` webhook questions, and the plain blind-overwrite lost-update question):** OOP, Design Patterns & Low Level Design, System Design & Architecture Basics, Concurrency & Locking

---

## Q1 — OOP, Design Patterns & Low Level Design

**Question:** Your team's `OrderService.placeOrder()` currently does everything inline:

```java
public void placeOrder(Order order) {
    orderRepository.save(order);
    emailService.sendConfirmation(order);
    inventoryService.decrementStock(order);
    analyticsService.recordPurchase(order);
}
```

A teammate proposes: "Instead of `OrderService` knowing about all three of these, let's have it just publish an `OrderPlacedEvent` and let each of those services listen for it independently."

- Is that the same fix as "inject an interface" (dependency inversion), or is it a genuinely different mechanism? What's the difference?
- In Spring, what are the two concrete pieces you'd need to wire this up — how does `OrderService` "publish," and how does something like `InventoryService` "listen"?
- What's a real downside this decoupling introduces — e.g. what happens if the inventory listener throws, or if it runs asynchronously?

**Candidate's answer** (given one hint per part after asking for a hint):
> "okay i think when pulish the notice the OrderService need to use some annotation to push the notification to specific topic and other class need to have annotaion Listener to listen in the topic to handle but i think when the InventoryService throw a exception the OrderService will not care or will not know the exception and it hardly debug and it can rollback if we add the @Transactional annotation"

**Feedback given:**
- Strong: correctly identified that a listener-style annotation is how the receiving side hooks into the event.
- Strong: correctly sensed that decoupling has a real cost around exception visibility.
- Missing: publishing isn't an annotation and there's no "topic" — that's messaging-system language, not Spring's in-process event mechanism. The publish side is `applicationEventPublisher.publishEvent(new OrderPlacedEvent(order))`; the listener side is `@EventListener` on a method taking the event type as its parameter.
- Missing: the DI-vs-event distinction wasn't answered — DI still calls a known collaborator directly (even behind an interface); with events, `OrderService` has zero knowledge of who's listening or how many listeners exist.
- Missing/misconception: the rollback claim has it backwards — by default a plain `@EventListener` runs synchronously, on the same thread and inside the same transaction as the publisher, so a thrown exception *does* propagate back and can roll back the whole `placeOrder()` transaction. The actually dangerous case is an `@Async` listener: it runs on a separate thread, `placeOrder()` already returned, and a thrown exception rolls back nothing and tells nobody — sync is coupled failure, async is invisible failure. Neither was distinguished.

**Score: 4/10**

---

## Q2 — System Design & Architecture Basics

**Question:** Your manager: "We need a simple personal TODO list API — add a task, mark it done, list your tasks. Nothing fancy."

Walk me through:
- What's the schema — the core table and its key fields?
- What are the endpoints, and which HTTP verbs/status codes would you use for each?
- Where would this simple design start to break down as usage grows — and is there a piece of it you'd deliberately *not* build yet, to avoid over-engineering a 3-endpoint service?

**Candidate's answer** (asked for clarification of what the last part of the question meant before answering):
> "so i think it should have task table and it has each field like this
> Id, name , duration, status , created_at
>
> i will design some endpoint like
>
> POST addTask status code : 201
> GET getTask status code: 200
> PUT editTask status code 200
> DELETE deleteTask status code 200
>
> GET listTask status code 200
> PUT markComplete status code 200
>
> so when i design the the app like this when have a real user i need to add the user table i need to add the authorization so the user A can not see the task list of user B"

**Feedback given:**
- Strong: correctly and unprompted identified that real users require a `user` table plus authorization so User A can't see User B's tasks.
- Strong: reasonable schema fields, correct `201` on create.
- Missing/misconception: endpoint naming uses verbs, not nouns (`addTask`, `getTask`, `editTask`, `markComplete`) — REST convention is the noun is the resource, the verb is the HTTP method: `POST /tasks`, `GET /tasks/{id}`, `GET /tasks`, `PATCH /tasks/{id}` (including marking complete via a partial update), `DELETE /tasks/{id}` (idiomatically `204`, not `200`, since there's no body). `PUT` implies a full replace; marking complete is a partial update, which is what `PATCH` is for.
- Missing: the actual bottleneck wasn't named — multi-tenancy/auth is a correctness gap needed on day one, not something that breaks "as usage grows." The scaling gap the question was fishing for is `GET /tasks` with no `page`/`size` params — unbounded as task count grows.
- Missing: "what would you deliberately not build yet" wasn't addressed at all.

**Score: 4/10**

---

## Q3 — Concurrency & Locking

**Question:** Two support agents open the same customer ticket at the same time. Agent A changes the status to "Resolved" and saves. A few seconds later, Agent B — still looking at their now-stale copy — changes the priority field and saves. The `Ticket` entity already has `@Version`.

- Walk me through what happens at the DB level when Agent B saves — what does `@Version` actually check, and what exception does JPA throw when that check fails?
- Should the app silently retry Agent B's save, or tell them "someone else changed this, please refresh"? Which is right here, and why?
- Is this the same shape of bug as "two users blindly overwrite each other's changes on a plain `UPDATE`," or does `@Version` change something structural about it?

**Candidate's answer:**
> "the version will hold the value of previous version so when agent B save it will compare with it and will handle if the distiguish more than 2 it will tell the app sliently retry because it will not tell the user B know the message or the type lock we use right now"

**Feedback given:**
- Strong: roughly right shape on the core mechanism — the loaded entity carries a version number, and the save has to check it against the current row.
- Missing: the exception was never named — `UPDATE ... WHERE id = ? AND version = ?` affecting 0 rows throws `ObjectOptimisticLockingFailureException` (or plain JPA `OptimisticLockException`).
- Missing/misconception: silently retrying is the wrong call here, and the reasoning matters — Agent B's retry would resubmit based on their stale view of the ticket, so B never learns Agent A also changed it. Silent retry only makes sense when reapplying the same operation on fresh data is safe (a stock decrement, an idempotent counter); for two humans editing the same record, surfacing "someone else changed this, please refresh" is correct.
- Missing: part 3 wasn't addressed — the structural difference is that a plain `UPDATE` with no version check always succeeds and silently clobbers the other person's write; `@Version` turns the same race into a *detected*, loud failure instead of an invisible one — it doesn't prevent the race, it prevents it from being silent.

**Score: 3/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | OOP, Design Patterns & Low Level Design (Observer / `ApplicationEvent`) | 4/10 |
| 2 | System Design & Architecture Basics (TODO API design) | 4/10 |
| 3 | Concurrency & Locking (JPA optimistic locking) | 3/10 |

**Overall: 3.7/10**

**Strengths:**
- Consistently reaches for the right *category* of fix without prompting (event-based decoupling, multi-tenant auth, version-based conflict detection) — the instinct for "what kind of tool solves this" is there.
- Willing to reason out loud through an unfamiliar mechanism rather than freeze or guess silently.

**Areas to review before the real interview:**
- REST resource modeling — nouns for paths (`/tasks`), HTTP verbs for actions; `PUT` (full replace) vs. `PATCH` (partial update) instead of inventing verb-named endpoints.
- Spring's actual event API — `ApplicationEventPublisher.publishEvent(...)` / `@EventListener`, and whether a listener runs synchronously (same transaction, exceptions propagate) or `@Async` (separate thread, failures go silent).
- Naming the exact exception, not just the mechanism — `ObjectOptimisticLockingFailureException` here; the same gap (concept roughly right, exact type/name missing) showed up in Q1 too.
- Scaling/bottleneck reasoning — for a "list" endpoint, the default answer to reach for is pagination (`page`/`size`) before anything else.
- Judgment calls on retry-vs-surface-to-user for a detected conflict — tie the decision to "does a human's mental model need correcting," not just "which mechanism is available."
