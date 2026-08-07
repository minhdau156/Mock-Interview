# Mock Interview — Junior Backend Developer
**Date:** 2026-08-07

**Categories covered (3-question session, seed-random start (seed=1207, start=8, step=1), categories 8/9/1 skipped because they repeat 2026-08-06's topics — Web Security, System Design, Core Java — per explicit request):** Tooling: Git, Build & Environment, OOP/Design Patterns & LLD, Databases: SQL, Indexing & Data Modeling

---

## Q1 — Tooling: Git, Build & Environment

**Question:** You open a PR for a bug fix. A reviewer leaves a comment: "I don't think this fix actually addresses the root cause — have you checked whether this can also happen on the batch-import path?" You're fairly confident your fix is correct and that batch import isn't affected. What do you do next, and how do you respond to the comment?

**Candidate's process:** Asked for a hint, then asked for a second hint ("how can I demonstrate it"), then said "I don't know what you mean" — no independent attempt was made even after two hints.

**Feedback given:**
- Missing: don't just reply with your opinion — go verify, then show your work.
  1. Check the code, not your memory: trace whether batch import calls the same method/logic you just fixed, or a separate implementation.
  2. Turn the check into evidence, not just a claim — point to the specific file/line so the reviewer can verify it themselves.
  3. Even better: write a quick test exercising the batch-import path with the same input that triggered the original bug — concrete proof, not an assertion, and it becomes a regression guard for free.
  4. If you're wrong, this process finds that out before you argue in the PR thread instead of after.
- Principle: a code review disagreement is resolved with evidence (code, tests, logs), not by restating your confidence level.

**Score: 2/10** — non-attempt even after two hints; lower than a typical "needed a hint" case.

---

## Q2 — OOP, Design Patterns & LLD

**Question:** You're reviewing a teammate's `Order` class (plain getters/setters for `items` and `status`) and an `OrderService.shipOrder(order)` method that loops over `order.getItems()` to compute a total, then checks `total > 0 && order.getStatus().equals("PAID")` before calling `order.setStatus("SHIPPED")`. What, if anything, is off about how this logic is split between the two classes — and how would you restructure it?

**Candidate's answer:**
> "i think in order service the function should have 1 functional itself like in the shipOrder it should just set status of order it should not calculate the totalPrice, we should seperate it to another function name calculatePrice with parameter is Order so we use this function in shipOrder to calculate the total, so it will make the method is easy to read and it should have 1 responsibility so it can easy to test and maintain"

**Feedback given:**
- Strong: correctly spotted the SRP smell in `shipOrder` doing two jobs (computing a total, then a status transition) — no hint needed.
- Missing: the deeper issue is *who should own this logic at all* — `OrderService` reaches into `Order` purely through getters and does the work itself, leaving `Order` as a passive data bag (an anemic domain model). The principle: **tell, don't ask** — move `calculateTotal()` and a `ship()` method (that enforces the paid/total invariant) onto `Order` itself, so `OrderService` just calls `order.ship()`.
- Extracting `calculatePrice(order)` as a helper inside `OrderService` (what was proposed) improves readability but still leaves the logic outside the class that owns the data — it doesn't close the encapsulation gap.

**Score: 4/10** — correct SRP instinct, no hint needed, but missed the core concept (tell-don't-ask / anemic domain model) the question targeted.

---

## Q3 — Databases: SQL, Indexing & Data Modeling

**Question:** Your team's `orders` table currently uses `DELETE FROM orders WHERE id = ?` on cancellation. Support just asked: "Can you restore an order a customer cancelled by mistake yesterday?" — and the answer right now is no. What would you change about how cancellation is handled, and what are the trade-offs of that approach?

**Candidate's answer:**
> "i think i will use the soft delete with the order... UPDATE orders SET status = 'CANCELED' WHERE id = ?, so when we need to restore we just set the status to pending again and it don't lost the data but the trade-offs it will make our DB more bigger so when the DB is big it can make the query slower so i think we need to delete the record have status CANCELED within 7 days if it not restore"

**Feedback given:**
- Strong: correctly identified soft delete via the existing `status` column, named a real trade-off (table growth), and proposed a retention policy rather than unbounded growth.
- Missing: the bigger practical trade-off is that every query against `orders` now must remember to filter out cancelled rows (`WHERE status != 'CANCELLED'`) — forgetting this anywhere silently surfaces cancelled orders as active. Proper indexing (partial index on active statuses) largely neutralizes the raw-size/performance concern.
- Missing: restoring by resetting `status` back to `"PENDING"` loses the order's actual prior state (e.g. it may have been `"PAID"`) — a separate `cancelled_at` timestamp alongside the lifecycle status preserves that information correctly.
- Missing: the proposed 7-day auto-purge re-introduces the original problem (a mistake noticed on day 8 is unrecoverable again) — fine as a deliberate, documented retention policy, but shouldn't be framed as a performance fix; performance is solved by indexing, retention is a separate business decision.

**Score: 6/10** — correct core mechanism and real trade-off awareness, but missed the sharper trade-offs (query-filtering burden, restore-data-integrity).

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Git Tooling — responding to PR feedback you disagree with | 2/10 |
| 2 | OOP/LLD — tell-don't-ask / anemic domain model | 4/10 |
| 3 | Databases — soft delete vs hard delete | 6/10 |

**Overall: 4.0/10**

**Strengths:**
- Genuinely correct instinct on Q2 (SRP smell in `shipOrder`) and Q3 (soft delete as the fix, plus a real trade-off), both without needing a hint.
- Reasoning is visible and clearly explained — walks through *why* each choice helps (testability, restorability), not just naming a keyword.
- Willingness to ask for a hint on Q1 rather than guessing blindly.

**Areas to review before the real interview:**
- **Q1 was a non-attempt even after two hints** — when stuck, give a best guess even under uncertainty ("I'm not sure, but I'd expect X because Y" scores far better than silence). The concept itself — resolve a code-review disagreement with evidence (trace the code path, write a test) rather than an assertion — is worth internalizing directly.
- **Tell-don't-ask / anemic domain models** (Q2) — a fresh gap. When a service method computes something purely from one object's own getters, ask "should that object do this to itself instead?"
- **Soft-delete trade-offs precision** (Q3) — the query-filtering burden and restore-data-integrity issues are sharper trade-offs than "the table gets bigger."
