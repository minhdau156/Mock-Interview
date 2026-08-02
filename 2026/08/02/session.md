# Mock Interview — Junior Backend Developer
**Date:** 2026-08-02

**Note:** Per user request, categories were still picked via the persona's normal seed mechanism (not deliberately targeted at the Priority List), but every question's concrete scenario/angle was constructed to be genuinely different from anything already written up in `KNOWLEDGE_SUMMARY.md` — category repeats are fine, repeated question content is not.

**Seed computation:** date-only formula (no time-of-day available): `day(2) × 137 + month(8) × 31 = 522`. `starting_index = (522 mod 10) + 1 = 3`. `step = [1,3,7,9][(522 ÷ 10) mod 4] = [1,3,7,9][0] = 1`.

**Categories:** Databases → Spring Boot & JPA/Hibernate → REST & API Design → Concurrency & Locking → Testing & Debugging.

---

## Q1 — Databases

**Question:** You're working on an e-commerce backend. The schema has `customers(id PK, email UNIQUE, ...)` and `orders(id PK, customer_id REFERENCES customers(id), total, ...)`. A teammate implementing a "delete my account" endpoint for a GDPR request runs `DELETE FROM customers WHERE id = 42;` and it throws a foreign key constraint violation. Their first instinct is to just drop the foreign key constraint so the delete "works." What's actually going on here, and how would you handle the account-deletion requirement instead?

**Candidate's answer:** it the relation between customer and order so when we delete the customer but the id that reference to it is existing so it can make this error; so we need to keep the reference key because it can help us to make the query easier; so before we delete the customer we need to check does any table have the column that have the reference key to the customer table; so we need to delete these record first because as we saw if the customer disappear the order should not exist in the DB; so we need to run the query DELETE FROM orders where customer_id = 41 first and then we write the query to delete the customer

**Score: 5/10**

**Strong:**
- Correctly diagnosed the cause — the `orders` row(s) with `customer_id = 42` still exist, so the FK constraint blocks the parent delete.
- Correctly rejected the teammate's shortcut of dropping the constraint.
- Good habit: checking whether *other* tables also reference `customers` before deleting, not just assuming `orders` is the only one.
- Right mechanical order for a manual cascade: handle child rows before the parent row.

**Missing:**
- The reasoning for *why* you keep the FK is off. "It makes the query easier" isn't the real reason — a foreign key enforces **referential integrity**: it guarantees an `orders` row can never point to a `customer_id` that doesn't exist. Drop it, and a customer delete can leave orphaned orders that silently break joins/refund logic/reporting later, with no error to flag it.
- The central point of this scenario: this is a **GDPR delete on an e-commerce order system**. Order/invoice records are usually financial records a business is legally required to *retain* even after the customer's account is gone. Hard-deleting the `orders` rows to satisfy the constraint likely violates that retention requirement. The real-world pattern is usually the opposite of what was proposed: don't cascade-delete the orders — anonymize the customer instead (scrub PII on the `customers` row, or point `orders.customer_id` at a "deleted user" placeholder / `ON DELETE SET NULL`), so order history survives with the PII removed.
- Worth knowing the schema-level options as named tools: `ON DELETE CASCADE`, `ON DELETE SET NULL`, `ON DELETE RESTRICT`.

**Tip:** Whenever "delete" and "financial/regulatory data" appear together, ask "am I allowed to actually remove this row, or do I need to anonymize instead?" before reaching for a cascade delete.

---

## Q2 — Spring Boot & JPA/Hibernate

**Question:** A teammate refactors a service so `placeOrder` calls `this.saveOrder(order)` directly (both methods in the same class), where `saveOrder` is `@Transactional` and calls `orderRepository.save(order)` then `inventoryRepository.decrementStock(order)`. In testing, a failure inside `decrementStock` (stock hit zero) does **not** roll back — `orderRepository.save(order)` stays committed even though the method threw. The teammate is confused: `@Transactional` is right there on `saveOrder`. Why doesn't the rollback happen, and what would you change?

**Candidate's answer:** i think because the exception DB is the checked exception so by default the Transactional annotation are rollback with the unchecked exception; so if we want to fix that we should mark rollbackFor = Exception.class property in Transactional annotation

**Score: 3/10**

**Strong:**
- Correctly recalled that `@Transactional`'s rollback behavior depends on exception type, and `rollbackFor = Exception.class` is the fix when a checked exception needs to trigger rollback.

**Missing:**
- That's the fix for a different bug than this one. Nothing in the scenario says `decrementStock` throws a checked exception — "stock hit zero" reads as a business-rule violation, typically unchecked. The rule was applied without checking whether its precondition actually held here.
- The actual root cause is **self-invocation bypassing the Spring proxy**. `@Transactional` works via a dynamic proxy wrapping the bean; that interception only happens when a call comes from *outside* the bean, through the proxy. `this.saveOrder(order)` is a plain call on the raw object, so the proxy — and the transaction it would have opened — is skipped entirely. `rollbackFor` changes nothing here, because no transaction is being managed on this call path at all.
- Real fix: move `saveOrder` into a separate injected bean so the call comes from outside the class and passes through the proxy.

**Tip:** Any time `@Transactional` "isn't working," first ask "is this method self-invoked (`this.method()` or a bare call from another method in the same class)?" before touching `rollbackFor`.

---

## Q3 — REST & API Design

**Question:** `GET /api/v1/orders?page=4000&size=25` is implemented as `SELECT * FROM orders ORDER BY created_at DESC LIMIT 25 OFFSET 100000`, with an index on `created_at`. Page 4000+ takes 8+ seconds while page 1 is instant. Why does a high page number get so much slower even with the index, and what would you change about the pagination approach?

**Candidate's answer:** because the OFFSET is scanning fulltable to offset 100000 record so when we have the index in created_at we use it to sort the result... but in this we don't have any condition in created_at we just use it to sort... so the index don't work, the DB will scan it record to count how many record i offset -> it is the full table scan so it will slow; but if we want to fix it we should keep the index in DB and custom the query to get the query base on created_at so we should use CURSOR_BASED on created_at: `SELECT * FROM orders WHERE created_at < ... LIMIT 25`, storing the cursor as the created_at value of the last record selected, so the index can work perfectly

**Score: 7/10**

**Strong:**
- Correctly identified that the slowness scales with `OFFSET` size, not table size in general.
- Landed on the right fix: keyset/cursor-based pagination, with essentially correct SQL and a correct description of storing the last row's value as the cursor.

**Missing:**
- "The index don't work" / "full table scan" overstates it. This typically still shows as an **Index Scan** in `EXPLAIN` — the engine uses the index to walk rows in order, but a B-tree only supports jumping to a key *value*, not to the Nth row by position, so it must walk and discard the first 100,000 entries. That's an `O(offset)` cost baked into offset pagination itself, not evidence the index isn't being used.
- Correctness gap in the cursor approach: if `created_at` isn't unique, `WHERE created_at < ?` alone can skip or duplicate rows across pages. The fix is a compound cursor/index on `(created_at, id)` to disambiguate ties.

**Tip:** Confirm theories about slow queries with `EXPLAIN`/`EXPLAIN ANALYZE` rather than guessing from symptoms alone — it tells you Index Scan-and-discard vs. true Seq Scan directly.

---

## Q4 — Concurrency & Locking

**Question:** Endpoint A (`processRefund`) locks `orders` then `inventory` with `SELECT ... FOR UPDATE`. Endpoint B (`receiveShipment`) locks `inventory` then `orders`. Under moderate concurrent load, a refund and a shipment-receipt touching the same order/inventory pair start throwing a database deadlock error and both requests fail. What's causing the deadlock specifically, and what would you change to prevent it?

**Candidate's answer:** with the moderate concurrent load it can have CCU between 50 to 500 so when we use the Pessimistic Locking... it lock the record the user1 active -> lock -> release it same time so when user500 active it will wait until the lock is released... so it can make the user5000 think it is DeadLock when it wait for a long time (timeout) so in this case i will use another lock is Optimistic Lock, i will use the @Version and check so it can prevent the DeadLock

**Score: 3/10**

**Strong:**
- Correctly recognized that pessimistic locking causes waiting under concurrent access.
- Correctly reached for optimistic locking (`@Version`) as an alternative strategy in the toolbox.

**Missing:**
- This conflates **lock contention/timeout** (many requests queueing for the same lock in one direction, eventually resolving) with an actual **deadlock**. The real cause is **lock-order inversion**, visible directly in the code: A locks `orders` then `inventory`; B locks `inventory` then `orders`. If both run at once, A holds `orders` and waits on `inventory` while B holds `inventory` and waits on `orders` — a genuine circular wait. The database's deadlock detector finds this cycle almost immediately (not after a long timeout) and kills one transaction outright.
- The standard, minimal fix: enforce one consistent lock acquisition order everywhere (e.g. always lock `orders` before `inventory`) — rewrite Endpoint B to match Endpoint A's order. With a single consistent order, a cycle can never form.
- Switching wholesale to optimistic locking might avoid this specific deadlock but is a bigger structural change than the problem calls for, and skips the actual lesson: any time two code paths lock more than one row, check whether all paths lock in the same order.

**Tip:** Whenever you see two-or-more `SELECT ... FOR UPDATE` calls in one transaction, check every other method locking the same tables for a different acquisition order — that's a deadlock waiting to happen regardless of load level.

---

## Q5 — Testing & Debugging

**Question:** A recursive function flattens a nested category tree. It's worked fine in production for months. Today, one customer's account (an unusually deep tree imported from a legacy system) triggers a `StackOverflowError` intermittently — fails maybe 1 in 5 times, works the other 4, with no code changes in between. Walk me through how you'd approach debugging this — including whether you'd reach for a debugger, print statements, or something else first, and why.

**Candidate's answer:** i think i will print the statements because it depend on the category because if the category and their children have the tree relationship it too much it will make the stack overflow error

**Score: 3/10**

**Strong:**
- Correctly identified that deep recursion on this account's tree is the underlying source of the stack overflow.

**Missing:**
- The answer only restated the obvious (deep tree → stack overflow) but never addressed the actual crux of the question: the tree is identical every single time, so why does it fail only 1 in 5 times instead of always or never? That's the part actually being tested.
- Missed explanation: this is a **resource/threshold bug**, not a logic bug. The recursion depth for this account's tree sits right at the edge of the JVM's available stack space. Small, incidental variations in how much stack the surrounding call chain has already consumed before reaching this method — which thread from the pool handled the request, how many filter/proxy layers ran first — tip it over the edge sometimes and not others.
- No debugging plan was given beyond "print statements," and no hypothesis for *why* it's intermittent was stated before picking a tool. A concrete plan would log the recursion depth reached (or the tree's actual depth) and compare it against the JVM's known default stack limit, to confirm the "right at the threshold" theory before touching the fix.
- No fix was proposed. The real fix for unbounded recursion depth is converting the function to an **iterative** traversal using an explicit stack (`Deque<Category>`) instead of the language call stack — this removes the dependency on JVM stack size entirely and scales to arbitrary tree depth. Raising `-Xss` only raises the threshold, it doesn't remove the fragility.

**Tip:** For any intermittent bug, form an explicit hypothesis for *why* it's intermittent before picking a tool — "is this timing/concurrency, or a resource/threshold effect (stack depth, memory, connection pool exhaustion)?" changes the whole debugging approach.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Databases — FK constraints, referential integrity & retention vs. cascade delete | 5/10 |
| 2 | Spring Boot/JPA — self-invocation bypassing `@Transactional` proxy | 3/10 |
| 3 | REST API — OFFSET vs. cursor/keyset pagination | 7/10 |
| 4 | Concurrency & Locking — deadlock (lock-order inversion) vs. lock contention | 3/10 |
| 5 | Testing & Debugging — intermittent bug from a non-timing (stack-depth) cause | 3/10 |

**Overall: 4.2/10**

**Strengths:**
- Once a cause is identified, reaches for the right high-level technique with working code — the cursor-pagination SQL was essentially correct, and the delete-children-before-parent ordering for the FK case was right.
- Comfortable committing to concrete answers and code rather than freezing under an unfamiliar scenario.
- Good instinct to check for other referencing tables before deleting a row (Q1).

**Areas to review before the real interview:**
- **Spring proxy mechanics — self-invocation.** `this.method()` calls bypass the AOP proxy entirely, so `@Transactional` (and any other proxy-based annotation) silently does nothing. A very common real-world gotcha; the answer defaulted to a previously-learned rule (checked/unchecked rollback) that didn't fit this bug — worth practicing reading the actual code before reaching for a memorized answer.
- **What a deadlock actually is.** A circular wait between transactions each holding what the other needs next — distinct from ordinary lock contention/queueing under load. Fix: consistent lock-acquisition ordering across all code paths, not a wholesale locking-strategy change.
- **Debugging intermittent bugs that aren't concurrency-timing related.** Build the habit of asking "why would identical input succeed sometimes and fail other times?" before reaching for a tool — resource/threshold effects (stack depth, memory, pool exhaustion) need a different framework than race-condition bugs.
- **Referential integrity vs. business/legal retention requirements.** Deleting child rows to satisfy a foreign key isn't automatically correct — check whether the child data needs to be retained (financial, legal, audit) and anonymize instead of cascading the delete when it does.
- **Precision on index behavior.** Distinguish "index used, but must walk and discard offset rows" from "full table scan" — matters for correctly reading `EXPLAIN` output rather than mislabeling the mechanism.
