# Mock Interview — Junior Backend Developer
**Date:** 2026-08-20

**Categories covered (3-question session, date-based seed (no timestamp available): seed=2988, start=9, step=7; sequence from start: System Design & Architecture Basics → Concurrency & Locking → Databases: SQL, Indexing & Data Modeling → Tooling: Git, Build & Environment → Testing & Debugging → ...; opening category (9) differs from 08/19's opener (2) per the no-repeat-opener rule; all three question scenarios were checked against every prior session (07/04 through 08/19) to avoid reusing a scenario/code snippet already asked):** System Design & Architecture Basics, Concurrency & Locking, Databases: SQL, Indexing & Data Modeling

---

## Q1 — System Design & Architecture Basics

**Question:** You're reviewing a PR for an "apply loyalty discount" endpoint:

```java
@RestController
public class OrderController {
    private final OrderRepository orderRepository;

    @PostMapping("/orders/{id}/apply-discount")
    public ResponseEntity<Order> applyDiscount(@PathVariable Long id) {
        Order order = orderRepository.findById(id).orElseThrow();
        if (order.getCustomer().getLoyaltyPoints() > 1000) {
            order.setTotal(order.getTotal() * 0.9);
        }
        orderRepository.save(order);
        return ResponseEntity.ok(order);
    }
}
```

There's no `OrderService` anywhere in this feature — the controller autowires `OrderRepository` directly, and the discount math lives right there in the endpoint method. It works, and the existing tests pass. What would you say about this in review, and why does the missing layer matter beyond just style?

**Candidate's answer:**
> "i think i will say that it need add the service layer it responsibility or the logic in it and the second answer i think it can make our code more easy to read and maintain or debug"

**Feedback given:**
- Strong: correctly named the core problem — the discount logic belongs in a service layer, not the controller.
- Strong: correctly named a real consequence (readability/maintainability/debuggability), even if stated generically.
- Missing: what a controller's actual job is — parse the HTTP request, call the service, map the result to a status code/body — nothing about loyalty points or discount math belongs there. Business rules leaking into the controller give it its own reason to change (a discount rule change) unrelated to its real job (an HTTP contract change).
- Missing: testability — right now, testing "does a 1000+ point customer get 10% off" requires spinning up the whole MVC layer (`MockMvc`, a fake `@PathVariable`, an `HttpServletRequest`) just to exercise a plain calculation. Move it into an `OrderService` method and it's a one-line unit test with no HTTP machinery involved.
- Missing: reusability — if a batch job, an admin tool, or a different endpoint later needs the same loyalty discount, that logic is stuck inside this one controller method and has to be duplicated. A service method is callable from anywhere.
- Missing: transaction boundaries — `@Transactional` idiomatically sits on the service method, not the controller — as this logic grows (discount + inventory check + audit log), you want one transactional unit, which the service layer is built for and the controller isn't.

**Score: 5/10**

---

## Q2 — Concurrency & Locking

**Question:** A teammate is building a `POST /webhooks/payment-confirmed` endpoint. The payment provider explicitly warns in their docs: "we may send the same webhook event more than once — your endpoint must handle duplicates." The teammate's first instinct is: "I'll just wrap the whole handler in `synchronized` so only one webhook can be processed at a time — that'll stop duplicates from being processed twice."

Would `synchronized` actually solve this? And separately — does this situation need a lock at all, or is there a simpler tool that solves it directly?

**Candidate's answer (after several clarifying nudges on what `synchronized` actually protects against, and on what a webhook retry looks like — a separate, later HTTP request rather than a concurrent one):**
> "i think the synchronized can not handle the same webhook events because it just usefull when have many thread but in this case it just have 1 thread but have many method that is duplicate when retries / so i think the way we can make is use the Idempotency key in this webhook url it can make the request handle properly"

**Feedback given:**
- Strong: after nudging, correctly identified why `synchronized` doesn't apply here — it protects against overlapping threads running at the same instant, not against two separate, sequential requests arriving seconds/minutes apart.
- Strong: landed on the right tool — an idempotency key.
- Missing: the initial instinct ("synchronized solves it easily... we don't need a lock" in the same sentence) showed the mental model wasn't solid yet — it took three rounds of nudging to separate "concurrent duplicate" from "retried duplicate."
- Missing: the actual mechanism — the webhook payload includes a unique event ID from the provider; the handler should atomically check-and-record that ID before doing the real work, typically a `UNIQUE` constraint on `event_id` in a `processed_webhook_events` table, with an insert that either succeeds (first time, proceed) or fails on the constraint (already seen, just return `200` again without reprocessing).
- Missing: the bigger lesson this question was testing — this is a "you don't need a lock at all" case; a DB-level unique constraint gives you the atomic guarantee for free, with no `synchronized`, no distributed lock, no extra infrastructure.

**Score: 5/10**

---

## Q3 — Databases: SQL, Indexing & Data Modeling

**Question:** A teammate designs the schema for a new "product tagging" feature:

```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  name VARCHAR(255),
  category_id BIGINT REFERENCES categories(id)
);
```

It ships, and a week later product marketing comes back with: "Actually, a product needs to be able to belong to *multiple* categories — e.g. a jacket should show up under both 'Outerwear' and 'Winter Sale'." The teammate's first reaction is: "I'll just add `category_id_2`, `category_id_3` as extra columns."

What's wrong with the original schema and with that fix, and how would you actually model this?

**Candidate's answer:**
> "i think i will make the relation between 2 table become n - n so when we add the field category_id_2 it will make our db more bigger / so we need to create the table that it will have 2 primary key of product and category / it like this / product_id reference products(id) / category_id reference categories(id); / so it can make our table clean"

**Feedback given:**
- Strong: correctly diagnosed the original schema's flaw — one `category_id` column can't represent "belongs to multiple categories."
- Strong: correctly rejected `category_id_2`/`category_id_3` as bloat that doesn't scale (there's always a category 4).
- Strong: correctly proposed the standard fix — a separate table linking `product_id` and `category_id`.
- Missing: the pattern has a name worth knowing — a **junction table** (or join table), the standard way to model many-to-many relationships.
- Missing: the two FK columns together should form a **composite primary key** — `PRIMARY KEY (product_id, category_id)` — not two separate primary keys. That composite key does double duty: row identifier *and* a free guarantee against tagging the same product into the same category twice.
- Missing: `category_id` should come out of `products` entirely — categories now live only in the junction table.
- Missing: one more concrete cost of the `category_id_2/3` approach — querying "all products in category X" becomes an ugly `OR` across multiple columns instead of one clean join; the junction table fixes the query shape too, not just the storage bloat.

**Score: 7/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | System Design & Architecture Basics — missing service layer (controller doing business logic) | 5/10 |
| 2 | Concurrency & Locking — idempotency key vs. `synchronized` for webhook duplicate delivery | 5/10 |
| 3 | Databases: SQL, Indexing & Data Modeling — many-to-many modeling via junction table | 7/10 |

**Overall: 5.7/10**

**Strengths:**
- Consistently lands on the correct core fix across all three topics (service layer, idempotency key, junction table), even when the first instinct needed correcting.
- Good instinct for spotting waste/bloat — flagged `category_id_2/3` as a scaling problem without being told.

**Areas to review before the real interview:**
- Naming the actual mechanism, not just the right-feeling answer — *why* a service layer helps (testability, reusability, transaction boundaries), not just "easier to read"; the term **junction table**; a DB `UNIQUE`/composite key as an atomic guarantee.
- Building a precise mental model of `synchronized` — it only blocks threads that are inside the block **at the same instant**; it does nothing for two separate, sequential requests (like a retried webhook). This took several nudges to untangle today.
- When unsure, reason from "what happens if two requests hit this at the exact same moment" *and* "what happens if the same request arrives again later" — they're different failure modes with different fixes.
