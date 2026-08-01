# Mock Interview — Junior Backend Developer
**Date:** 2026-08-01

**Note:** This session was deliberately scoped to avoid topics/angles already documented in `KNOWLEDGE_SUMMARY.md` (per user request) — categories were still picked via the persona's normal seed mechanism, but each question targeted a sub-angle flagged there as "not yet tested," rather than re-drilling previously scored ground.

**Seed computation:** date-only formula (no time-of-day available): `day(1) × 137 + month(8) × 31 = 385`. `starting_index = (385 mod 10) + 1 = 6`. `step = [1,3,7,9][(385 ÷ 10) mod 4] = 7`.

**Categories:** Concurrency & Locking → Databases → Tooling (Git/Build/Environment) → Testing & Debugging → Spring Boot & JPA/Hibernate.

---

## Q1 — Concurrency & Locking

**Question:** Your team runs the order service on two app instances behind a load balancer for redundancy. A teammate "fixes" a duplicate-processing bug by adding `synchronized` to the method that processes an incoming order webhook. In staging (one instance) it works perfectly. In production (two instances) you still occasionally see the same order processed twice. Walk me through why `synchronized` didn't actually solve this in production, and what you'd reach for instead.

**Candidate's answer:** i think it can be possible there are mutiple server so it process twice or more and it duplicate the order or the user is retry when our server missed the idempotency key so it can hit twice when the user retry

*(Candidate first asked "can you explain what synchronized are used to do" — given a definition of the intrinsic-lock mechanism before answering.)*

**Score: 5/10**

**Strong:**
- Correctly landed on the actual root symptom — multiple servers is exactly why duplicates can still slip through.
- Brought in idempotency/retry as a solution angle unprompted — genuinely good, practical instinct.

**Missing:**
- The mechanism: a `synchronized` lock is tied to an object living inside **one JVM process**. Instance A and Instance B are two separate JVMs, each with its own independent copy of that lock — Instance A's lock has zero awareness of Instance B's. A thread on server 1 and a thread on server 2 can each acquire "their own" lock and enter the method at the exact same instant. `synchronized` only ever coordinates threads *within a single process*.
- A hint was needed to recall what `synchronized` does before reasoning about the failure.
- Full fix menu: a **distributed lock** (Redis `SET key value NX PX <ttl>` — works across instances because Redis is a shared external coordinator), **or**, often cleaner, remove the need for a lock via **idempotency** — a DB unique constraint on the order's external/webhook ID, or an idempotency key checked atomically before processing.

**Tip:** Reflex for "works on staging (1 instance), breaks in prod (2+ instances)": ask "is this in-memory coordination, or does it need a shared source of truth outside any single process?"

---

## Q2 — Databases

**Question:** A teammate adds a composite index `(customer_id, status, created_at)` on the `orders` table to speed up the "recent orders for a customer" query. A few weeks later, someone complains that a different query — `SELECT * FROM orders WHERE status = 'PENDING'` — is doing a full table scan and is slow, even though `status` is right there in that same composite index. Why doesn't the existing index help that second query, and what would you tell them?

**Candidate's answer:** it have the full scan is correct because the composite index is structure base on the first column we add into this index; like this it will sort the first column (customer_id) if 2 then sort the status and finally sort the created_at; so if we have some query it will be read at customer_id first to see and will reflect the status later and finally created_at column; so in the query like this we just find it in the status column so it will look up in the status query but the composite index will work if this query having the condition in customer_id; without it it will scan full table and make the query slower; so if we want to improve we need to know does this request are used recently or the query is important if it is we should add the index in status so that the index can work effectively

**Score: 8/10**

**Strong:**
- Correctly explained the B-tree composite index ordering mechanism: sorted by first column, then second within it, then third — so a query skipping the leading column can't use the sort order.
- Correctly distinguished which queries benefit vs. which don't.
- Good practical judgment — deciding whether to add a new index based on query importance/frequency, rather than reflexively indexing everything.

**Missing:**
- The formal name: **leftmost-prefix rule** — a composite index on `(a, b, c)` serves queries filtering on `a`, `a+b`, or `a+b+c`, but never `b` alone.
- The write-side cost of adding another index: every index speeds up reads but adds overhead to every `INSERT`/`UPDATE`/`DELETE` on the table.
- Habit to add: run `EXPLAIN`/`EXPLAIN ANALYZE` to confirm the full/sequential scan before deciding to add an index.

**Tip:** Write out "sorted by col1 → then col2 within col1 → then col3 within col2" every time — that visual makes leftmost-prefix obvious.

---

## Q3 — Tooling: Git, Build & Environment

**Question:** You deploy your Spring Boot service to staging for the first time. It boots fine on your laptop, but in staging it crashes on startup with something like `Could not resolve placeholder 'DB_PASSWORD'` — a value that's clearly sitting right there in your local `application.properties`. What's actually going on here, and how would you set the project up so this doesn't keep happening as it moves from your machine → staging → prod?

**Candidate's answer:** i think we should divide each variable base on specific enviroment because when set variable we should use .env in local , .env.staging in staging and .env.production in production

*(Candidate asked for more explanation of the question first — clarified the two parts being asked: the immediate cause, and the durable fix.)*

**Score: 6/10**

**Strong:**
- Good instinct that config needs to be environment-specific, not a single shared setup.
- Naming a concrete convention (separate files per environment) shows systematic thinking.

**Missing:**
- The actual "why did it work on my laptop" root cause: something local (a `.env` file, IDE run config, exported shell variable) is quietly supplying `DB_PASSWORD`, and it never got replicated to staging.
- Spring's own mechanism for this: **profiles** (`application-dev.properties` / `application-staging.properties` / `application-prod.properties`, via `spring.profiles.active`) — good for non-secret environment differences.
- For the actual secret, the stronger production pattern is the deployment platform injecting the real value as a runtime environment variable (CI/CD secret store, Kubernetes Secret, Secrets Manager) rather than a checked-in `.env.staging` file — `.env` files must stay in `.gitignore` the moment they hold a real secret; commit a `.env.example` with placeholders instead.

**Tip:** Split it into two lenses: "config that differs by environment but isn't secret" → Spring profiles; "actual secrets" → platform-injected env vars / secret manager, never a repo file.

---

## Q4 — Testing & Debugging

**Question:** A support ticket comes in: "Customer cancelled order #4821 after it had already shipped, and the system still processed a refund — it shouldn't have, since shipped orders aren't refundable." You open the code and can see roughly where the cancel logic lives. Before you touch a single line of the fix, what's the very first thing you do, and why does doing it in that order matter?

**Candidate's answer:** so i will check the test first to see so it can have the problem when someone refactor the logic code but forgot to run the test or something else so it can happen so before fix we should run the test to see and we don't need to rewrite the test

*(Candidate requested two hints; still landed on a materially different/incorrect interpretation — rerunning existing tests rather than writing a new failing test.)*

**Score: 3/10**

**Strong:**
- Testing is clearly on the candidate's radar as part of the process.

**Missing:**
- The core answer: write a **new failing test first** that encodes the exact bug (e.g. `cancelOrder_whenAlreadyShipped_doesNotTriggerRefund()`), since there's almost certainly no existing test covering this case (otherwise the bug couldn't have shipped). Run it against current code — it should **fail**, proving the bug is reproduced. Fix the code, rerun — it should **pass**, proving the fix works.
- Why order matters: a failing test *before* the fix proves you reproduced the real bug; a passing test *after* proves the fix addresses it; the test then stays in the suite permanently as a regression guard.
- Don't skip writing the test — it's the deliverable that proves the bug is actually fixed.

**Tip:** Reflex for any bug fix: reproduce → write a failing test that encodes the bug → fix the code → watch it go green → keep the test. Practice deliberately on the next 2-3 real bugs.

---

## Q5 — Spring Boot & JPA/Hibernate

**Question:** You build an admin endpoint, `GET /api/orders`, that returns all orders along with each order's customer name. It works fine in testing with 10 sample orders. After launch, with a few thousand orders in the table, the endpoint gets noticeably slower — and when you check the query logs, you see one query for the order list, immediately followed by a huge burst of nearly identical individual queries. What's happening here, and what's one way you'd fix it?

**Candidate's answer:** it the N + 1 query problem; when i call to db to get the orders i want to get all orders but when i use this query it vill relevant with the User and it can call the query to select the user having this customer_id; so if i the orders have 10000 record it will call after that 10000 query just to get the order's customer name so we just need 1 query but it took 10001 query so it will down our db, our server if CCU is higher; so if we want to fix it we need to use each approach JOIN FETCH in Spring data jpa; we join 2 table order and user together i select the order and customer name at once (big query)

**Score: 8/10**

**Strong:**
- Correctly named the problem immediately — N+1 — and explained the mechanism accurately: 1 query for the list, then a separate query per order to resolve its customer.
- Correctly reasoned about impact — DB load, and how it compounds under concurrent users.
- Correct standard fix: `JOIN FETCH` in Spring Data JPA.

**Missing:**
- The *why*: the `Order → Customer` association is almost certainly **lazy-fetched** — Hibernate fires a query for it only when accessed, once per order. Naming "lazy loading" explicitly ties symptom to cause.
- Two other standard fixes worth knowing: `@EntityGraph` and batch fetching (`@BatchSize`).

**Tip:** Reflex for spotting N+1 in logs: one query for a list, immediately followed by a burst of near-identical queries differing only by one ID.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Concurrency & Locking — synchronized across multiple instances | 5/10 |
| 2 | Databases — composite index leftmost-prefix rule | 8/10 |
| 3 | Tooling — env config across dev/staging/prod | 6/10 |
| 4 | Testing & Debugging — test-driving a bug fix | 3/10 |
| 5 | Spring Boot/JPA — N+1 query problem | 8/10 |

**Overall: 6.0/10**

**Strengths:**
- Strong first-principles reasoning on DB indexing (composite index mechanics) and immediate, accurate recognition of the N+1 problem with a correct fix.
- Good real-world instincts volunteered without prompting — idempotency as a fix direction (Q1), environment-specific config awareness (Q3).
- Comfortable reasoning out loud and asking for hints instead of freezing or guessing blindly — keep doing this in the real interview.

**Areas to review before the real interview:**
- **Cross-instance coordination**: `synchronized`/in-process locks only coordinate threads within one JVM — zero reach across separate app instances. Review distributed locking (Redis `SET NX PX`) and idempotency-based fixes (DB unique constraints, idempotency keys).
- **Config vs. secrets separation**: Spring profiles handle non-secret environment differences; real secrets should be injected by the deployment platform at runtime — not distributed as per-environment files, even gitignored ones.
- **Test-driving a bug fix** (weakest answer this session, needed heavy support): write a *new* test that encodes the exact bug and watch it fail first, before touching the fix.
- Minor polish: name the **leftmost-prefix rule** explicitly for composite indexes, and name **lazy loading** explicitly as the root cause of N+1 — both were reasoned correctly but left unnamed.
