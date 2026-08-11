# Mock Interview — Junior Backend Developer
**Date:** 2026-08-11

**Categories covered (3-question session, seed-random start (seed=1755, start=6, step=9), categories 5/8/9 skipped because they repeat 2026-08-10's topics — REST & API Design, Web Security, System Design — sequence otherwise: Concurrency & Locking → Spring Boot & JPA/Hibernate → Databases: SQL/Indexing/Data Modeling):** Concurrency & Locking, Spring Boot & JPA/Hibernate, Databases: SQL, Indexing & Data Modeling

---

## Q1 — Concurrency & Locking

**Question:** You're building the checkout flow for a flash sale where exactly 1 unit of a hot item is left in stock. Two customers click "Buy" within the same second. Your tech lead says: "For this one specifically, don't use optimistic locking — use `SELECT ... FOR UPDATE`."

Why would pessimistic locking be the right call here instead of the `@Version`-based approach you might reach for elsewhere? Walk me through what actually happens at the DB level when both requests hit at the same instant, and what could go wrong if you picked the other approach for this scenario.

**Candidate's answer:**
> "i think we choose the Pessimisstic Lock in this case because it can lock the record and just 1 thread can update the record of DB / Optimistic Lock use the @Version it will check the previous version with the other version if the version is just different about 1, it will update otherwise it will choose the one have the @Version is satified and other need to be retry / so in this case when it just have 1 unit of hot item we should use the Pessimistic Lock because i think it will lock the DB and it is safe and the request is rarely need to retry if it out of stock we will pop up the message"

**Feedback given:**
- Strong: correctly chose pessimistic locking for this scenario; roughly correct grasp of the optimistic-locking version-check/retry mechanism.
- Missing: precision on scope — `SELECT ... FOR UPDATE` locks the row, not "the DB."
- Missing: the actual behavior of the losing side — it blocks/waits for the lock to release, it doesn't fail-and-retry; that's the real distinction from optimistic locking.
- Missing: the efficiency argument for why pessimistic wins here specifically — with only 1 unit and heavy contention, optimistic locking's losers do a full wasted read-modify-write-retry cycle that's almost certainly doomed anyway; pessimistic avoids that wasted work entirely.
- Missing: pessimistic's own downside — a pile-up of waiting transactions can cause timeouts, and locking multiple rows in inconsistent order across code paths risks deadlock.

**Score: 6/10**

---

## Q2 — Spring Boot & JPA/Hibernate

**Question:** A teammate wrote a batch import job:

```java
List<Product> products = csvParser.parseAll(file);         // Product is @Entity, no equals()/hashCode() overridden
Set<Product> uniqueProducts = new HashSet<>(products);
productRepository.saveAll(uniqueProducts);
```

Two rows in the CSV clearly have the same SKU, but after this runs, both still end up saved as separate rows anyway — the dedup didn't work.

What's actually happening here, and what would you change on the `Product` entity to fix it — keeping in mind these `Product` objects haven't been persisted yet, so they don't have an `id` assigned?

**Candidate asked three clarifying questions** ("what is SKU," "what is dedup," "what does 'both still end up saved as separate rows anyway' mean") **and needed two hints** (what Java gives you by default when `equals`/`hashCode` aren't overridden; to prompt for the actual fix) before answering.

**Candidate's answer:**
> "ukm i think if i don't override 2 function it will check 2 row but different object so it insert 2 time" ... "i think i will override 2 function and add the id"

**Feedback given:**
- Strong: with a hint, correctly identified that unoverridden `equals()`/`hashCode()` fall back to identity-based defaults, which is the real root cause.
- Missing: the proposed fix (`equals`/`hashCode` on `id`) doesn't work here — every `Product` is transient, so every `id` is `null`; comparing on `id` would make all transient products (even different SKUs) look equal to each other, silently collapsing distinct products rather than just fixing the dedup.
- Missing: connection to a prior lesson — this is the same category of mistake as a Spring/JPA question from two sessions ago (relying on `id` for `equals`/`hashCode` without checking it's actually populated at that point in the object's lifecycle).
- Missing: the actual correct fix — override `equals()`/`hashCode()` on the SKU (the real, always-populated business identity), not `id`.

**Score: 4/10**

---

## Q3 — Databases: SQL, Indexing & Data Modeling

**Question:** Marketing asks for a report: "every customer, and how many orders they've placed — including customers who haven't ordered yet." A teammate writes:

```sql
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.discount_code = NULL
GROUP BY c.name;
```

When you run it, two things are wrong: customers with zero orders are missing entirely from the report, and the `discount_code` filter isn't excluding anything — even though you know for a fact there are orders with no discount code.

What's wrong with each part, and how would you fix the query?

**Candidate's answer:**
> "i think first they use different JOIN / they should use the LEFT JOIN so this can get the customer who don't have any order yet and we can count it in this / and the second one is apply the discount_code so the target they want to check all the order that include either discount or not so we need to remove it / so the good query is like that: SELECT c.name, COUNT(o.id) AS order_count FROM customers c LEFT OUTER JOIN orders o ON c.id = o.customer_id GROUP BY c.name;"

**Feedback given:**
- Strong: correctly diagnosed the `INNER JOIN` vs `LEFT JOIN` issue with no hint needed, and produced a fully correct rewritten query.
- Missing: didn't explain *why* `= NULL` is broken — comparing anything to `NULL` (including `NULL` itself) evaluates to `UNKNOWN` under SQL's three-valued logic, and `WHERE` treats `UNKNOWN` as `false`, so `x = NULL` never matches any row. The dedicated fix for a real null-check is `IS NULL`/`IS NOT NULL`, not plain `=`; here it happened to be unrelated scope creep, so dropping it was the right call, but the reasoning behind *why* should be explicit.

**Score: 7/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Concurrency & Locking — pessimistic vs. optimistic locking for a last-unit flash-sale checkout | 6/10 |
| 2 | Spring Boot & JPA/Hibernate — `HashSet` dedup failing on entities with no `equals`/`hashCode` override; danger of using `id` pre-persist | 4/10 |
| 3 | Databases — `INNER` vs `LEFT JOIN` for a "customers with zero orders" report, and the `= NULL` vs `IS NULL` pitfall | 7/10 |

**Overall: 5.7/10**

**Strengths:**
- Q3 showed real improvement in speed — diagnosed the JOIN bug immediately with zero hints and produced a fully correct rewritten query.
- Consistently reasons in the right direction even when imprecise (locking mechanism, HashSet mechanics) — the gaps are in stating the exact mechanism, not in the overall judgment.

**Areas to review before the real interview:**
- Locking: always name exactly what's locked (row, not "the DB") and what the losing side actually does (blocks and waits, vs. fails and retries).
- Entity keys: this is the second session in a row hitting the same trap — using an entity's `id` for `equals`/`hashCode` without checking whether it's actually populated at the point the object is used as a key. Ask "is `id` guaranteed non-null here?" before reaching for it; use a stable business key (SKU, email, etc.) when it isn't.
- SQL NULLs: internalize that `= NULL` is dead code — it filters out every row, including ones that are actually `NULL` — and `IS NULL`/`IS NOT NULL` are the only correct tools for null checks.
