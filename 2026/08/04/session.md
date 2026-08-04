# Mock Interview — Junior Backend Developer
**Date:** 2026-08-04

**Note:** Per user request at session start, this session was shortened to 3 questions instead of the usual 5, and every question's concrete scenario was required to be new ground versus `KNOWLEDGE_SUMMARY.md` — categories still picked via the seed-randomized algorithm (seed 796, start index 7, step 9), but sub-topics within each category were deliberately steered away from angles already deeply covered (e.g. the classic counter race-condition and the timing-bug debugging framework, both already at 8/10 in prior sessions) toward angles explicitly flagged as untested.

**Categories:** Testing & Debugging → Concurrency & Locking → REST & API Design.

---

## Q1 — Testing & Debugging

**Question:** You're reviewing a teammate's PR. They added this test for a discount service:

```java
@Test
void testDiscount() {
    Order order = new Order();
    order.addItem(new Item("Widget", 100.0));
    order.addItem(new Item("Gadget", 50.0));
    order.applyCoupon("SAVE10");
    assertTrue(order.getTotal() < 200.0);
    assertNotNull(order.getAppliedCoupon());
    assertEquals(2, order.getItems().size());
}
```

You've noticed this test has failed intermittently in CI — maybe 1 in 20 runs — with no code changes, and nobody can reproduce it locally. Also, `order.getTotal() < 200.0` would already have passed even with zero discount applied — the raw pre-discount total is $150. What would you say in the PR comment, and what would you ask the author to change?

**Candidate's answer:** (asked several clarifying questions — which assertion fails intermittently, what `applyCoupon("SAVE10")` actually does, and whether `order` was a freshly created object in this test method — then, after one hint pointing at the weak assertion and the multiple behaviors bundled into one test, said "i don't have any idea" and asked for the answer directly without attempting one.)

**Score: 3/10**

**Strong:**
- Good instinct to gather facts before answering — asking which assertion failed, what `applyCoupon` actually does, and whether the `Order` was fresh, is exactly how you should approach an ambiguous bug report in real life.

**Missing:**
- **One test, three behaviors.** `testDiscount` checks the discounted total, that a coupon got stored, and that the item count is 2 — three different behaviors in one method with a name that doesn't say which. If it fails, you can't tell which behavior broke without reading the stack trace closely. Fix: split into separate tests, e.g. `appliesCoupon_reducesTotalByExactPercentage()` and `appliesCoupon_storesAppliedCouponOnOrder()` — item count doesn't even belong in a discount test at all.
- **The assertion doesn't prove what it claims to.** `assertTrue(order.getTotal() < 200.0)` would pass even if `applyCoupon` did *nothing* — pre-discount total is already $150, well under $200. Fix: assert the exact expected value — `assertEquals(135.0, order.getTotal(), 0.001)` (arrange 150 → act 10% off → assert exactly 135, with a small tolerance for floating point).
- **The flaky count (2 vs 3) with zero code changes and no local repro is a test-isolation smell, not a coincidence.** A brand-new `new Order()` shouldn't ever end up with an extra item from code this method never calls — the likely cause is shared mutable state leaking across tests (a static/shared item catalog, a reused fixture, a test database not rolled back between runs, or parallel test execution touching shared state). Fix is to find and eliminate the shared state, not to loosen the assertion or re-run until green.
- **Why this matters, worth saying explicitly:** a flaky test is worse than no test, because after enough "ignore that one, it's just flaky" moments, real failures start getting ignored too — CI stops being a signal anyone trusts.

**Tip:** Next time you're stuck, try stating your best guess out loud even if you're unsure — "I think the assertion looks weak because..." scores better than "I don't know," and it's often enough to get you 80% of the way there yourself.

---

## Q2 — Concurrency & Locking

**Question:** A teammate wrote this to cache product lookups and avoid hitting the DB on every request:

```java
@Service
public class ProductService {
    private final Map<Long, Product> cache = new HashMap<>();

    public Product getProduct(Long id) {
        if (cache.containsKey(id)) {
            return cache.get(id);
        }
        Product product = productRepository.findById(id).orElseThrow();
        cache.put(id, product);
        return product;
    }
}
```

This is a `@Service`, so Spring manages it as a singleton shared across all incoming request threads. Under load testing with many concurrent requests, they're occasionally seeing `ConcurrentModificationException` and sometimes just plain wrong/missing entries in the cache — no exceptions at all. What's going on here, and how would you fix it?

**Candidate's answer:** (asked what `ConcurrentModificationException` means, then answered:) it happen because like you say the current request used the Map simutanouesly so it will detect and throw the Exception it have the wrong/missing because under concurrent request it will update the DB and get in the cache so it will inconsistency between 2 of them and i think if i want to fix it i should have the Invalidation in the cache and i will use some lock to lock the Map

**Score: 5/10**

**Strong:**
- Correctly identified the core cause: multiple request threads hitting the same shared `HashMap` at the same time, with no synchronization protecting it.
- Locking as a fix direction is on the right track — mutual exclusion does solve the underlying problem.

**Missing:**
- The "wrong/missing entries" symptom isn't a DB-vs-cache inconsistency (nothing here is going stale — no TTL, no external update event). It's `HashMap` internally corrupting its own structure under concurrent `put()` calls: `put()` can trigger a resize/rehash of its internal bucket array, and if two threads hit that at the same instant, they race on the same internal array — silently dropping/overwriting an entry, or a concurrent `get()` returning `null` for a key that should be there — with zero warning, because plain `HashMap` makes no thread-safety guarantees at all.
- "Cache invalidation" isn't the right tool here — there's no staleness problem to invalidate against. The fix is purely about thread-safety of the shared collection.
- Manually locking the whole `Map` with `synchronized` would work, but it's not the idiomatic first answer for this shape. The standard fix is `ConcurrentHashMap`, which handles concurrent reads/writes internally (lock striping / CAS) without hand-rolled locking:
```java
private final Map<Long, Product> cache = new ConcurrentHashMap<>();

public Product getProduct(Long id) {
    return cache.computeIfAbsent(id, key -> productRepository.findById(key).orElseThrow());
}
```
- Bonus bug worth naming: `containsKey` → `get` → `put` is itself a non-atomic check-then-act — two threads can both miss the cache and both hit the DB. `computeIfAbsent` makes that atomic per key.

**Tip:** Whenever you see a plain mutable `Map`/`List`/`Set` as a field in a Spring singleton bean touched by multiple request threads, the reflex should be "is there a `java.util.concurrent` type that already solves this" before reaching for your own `synchronized` block.

---

## Q3 — REST & API Design

**Question:** You're designing the API for an e-commerce system. A teammate proposes `GET /orders?customerId=42` to list a customer's orders, but another teammate insists it should be `GET /customers/42/orders`. Both return the same data. Which would you choose, and what's your reasoning — and would your answer change if the request was "get all orders placed in the last 7 days across all customers"?

**Candidate's answer:** i think i will keep the second one for my answer because we should keep the id of customer in the body of URL bot in the parameter the parameter i think it shoud use for filter or something like get the range time and my answer will change if the request is get all orders placed in the last 7 days across i will use the first one with another parameter GET /orders?range=7D; i think it should like or if we want to get the orders or specific customer in the last 7 days we can mix to of them like this GET /customers/42/orders?range=7D

**Score: 8/10**

**Strong:**
- Correctly chose the nested route (`/customers/42/orders`) for a single customer's orders, with the right instinct: the customer identity belongs in the path because it's a required part of *which* resource collection you're addressing, not an optional filter.
- Correctly recognized that a query parameter is the right tool for the date-range filter, not another path segment.
- Correctly changed the design when the scope changed to "all customers, last 7 days" — recognized that nesting under a specific customer no longer makes sense once the query spans everyone, and reverted to the flat `/orders?range=7D`, with no hint needed.
- Bonus: correctly composed both — `/customers/42/orders?range=7D` — for "this customer's orders in the last 7 days."

**Missing:**
- The general rule was demonstrated through examples but never stated as a reusable heuristic: path parameters identify which specific resource/sub-collection you're scoped to; query parameters filter or modify the result set within that scope.
- Terminology was a little loose ("keep the id... in the body of the URL") — the correct term is **path parameter** (or path segment), not "body."
- Minor precision point: `range=7D` isn't a self-documenting/standard query param shape — a real API would more typically use `?since=2026-07-28` (an explicit date) or `?days=7`.

**Tip:** When justifying a REST design choice out loud, state the general rule first ("path scopes the resource, query filters it"), then show the example — interviewers score the principle, not just the correct-looking URL.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Testing & Debugging — flaky test diagnosis, weak assertions, one-behavior-per-test | 3/10 |
| 2 | Concurrency & Locking — thread-unsafe `HashMap` under concurrent access | 5/10 |
| 3 | REST & API Design — nested vs. flat routes, path vs. query parameters | 8/10 |

**Overall: 5.3/10**

**Strengths:**
- Sharpest reasoning of the session on REST resource modeling — correctly reversed the design decision when the scope of the request changed (single customer → all customers), with no hint needed.
- Good instinct to ask clarifying questions before committing to an answer (Q1, Q2) rather than guessing blind.
- Correctly diagnosed the concurrency root cause (unsynchronized shared `HashMap` across request threads) even though the fix reached for wasn't the most idiomatic one.

**Areas to review before the real interview:**
- **Flaky test root causes and test hygiene.** Practice naming test-isolation/shared-state as the cause of a flaky test with no code changes and no local repro (not a coincidence); tighten assertions to exact expected values instead of loose bounds; split multi-behavior tests into one-behavior-per-test.
- **Thread-safe collection alternatives.** `ConcurrentHashMap` (and `computeIfAbsent` for atomic check-then-compute) is the standard fix for a shared mutable map touched by multiple threads — prefer it over hand-rolled `synchronized` locking; know that plain `HashMap`'s internal resize/rehash is what actually corrupts state under concurrent `put()` calls.
- **REST design vocabulary.** State the path-vs-query-parameter rule explicitly as a principle, not just through a correct example — and use precise terms ("path parameter," not "URL body").
