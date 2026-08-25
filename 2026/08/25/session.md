# Mock Interview — Junior Backend Developer
**Date:** 2026-08-25

**Categories covered (3-question session, date-based seed (no timestamp available): seed = 25×137 + 8×31 = 3673, start=4, step=9; sequence: Spring Boot & JPA/Hibernate → Databases: SQL, Indexing & Data Modeling → OOP, Design Patterns & Low Level Design; opening category (4) differs from 08/23's opener (10) per the no-repeat-opener rule. Note: Q1's scenario (`LazyInitializationException` from a lazy collection touched after the transaction/session closes) was not caught as a repeat during pre-check — it duplicates 08-05's Q1, which used an older `## Question N —` header format that the pre-check's header search missed. Q2 and Q3 were confirmed fresh against every prior session.):** Spring Boot & JPA/Hibernate, Databases: SQL, Indexing & Data Modeling, OOP, Design Patterns & Low Level Design

---

## Q1 — Spring Boot & JPA/Hibernate

**Question:** A teammate writes this endpoint:

```java
@GetMapping("/customers/{id}")
public Customer getCustomer(@PathVariable Long id) {
    return customerRepository.findById(id).orElseThrow();
}
```

`Customer` has `@OneToMany(mappedBy = "customer", fetch = FetchType.LAZY) private List<Order> orders;`. It works fine as-is. Then another teammate adds one debug line right after the fetch, inside the same controller method:

```java
log.info("orders count: {}", customer.getOrders().size());
```

Now the endpoint throws:

```
org.hibernate.LazyInitializationException: failed to lazily initialize a collection of role: ...Customer.orders, could not initialize proxy - no Session
```

Nothing else changed. Why does adding a harmless-looking log line break this, and what's the actual fix — not just "make it eager"?

**Candidate's answer** (asked for two hints; first nudged toward what owns the DB session/transaction, second toward whether the session was still open at `.size()`):
> "i think when connect to DB i will open a transaction about that so when the transaction close and we call the size() of thethis it will get the error / so i think we should fetch it as one so when we need we can still use it without call it and refetch i will make the problem like this again"

**Feedback given:**
- Strong: correct core mechanism after a hint — the session/transaction tied to `customer` is closed by the time `.getOrders().size()` runs, and a lazy proxy can't hit the DB once its session is gone.
- Missing: why this specific endpoint — no `@Transactional` is declared anywhere; a Spring Data repository method like `findById` is transactional only for its own call, so the transaction commits and the session closes the instant `findById` returns. `customer` comes back detached; the log line is just the first place anyone touches `orders`, not the cause.
- Missing: the question explicitly ruled out "just make it eager," and that's what was proposed — needed the reasoning for why blanket eager fetching is the wrong general fix (loads `orders` on every fetch everywhere, including call sites that never need it).
- Missing: concrete correct fixes — wrap the real business logic in a `@Transactional` service method and return a DTO (not the entity) to the controller; or use a repository query with `JOIN FETCH`; or project straight to a DTO in the repository so the controller never touches a lazy proxy.

**Score: 4/10**

---

## Q2 — Databases: SQL, Indexing & Data Modeling

**Question:** A teammate designs this table for a new "customer contact info" feature:

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    phone_numbers VARCHAR(255)  -- e.g. "555-1234,555-5678"
);
```

Their reasoning: "Most customers only have one or two numbers, so cramming them into one column is simpler than a whole extra table." It ships. A few weeks later, product asks: "let support search for a customer by any of their phone numbers." The teammate writes `WHERE phone_numbers LIKE '%555-1234%'` and it works, but isn't happy with it.

What's fundamentally wrong with the original design, why does it bite specifically on this new feature, and how would you actually model it — what's the trade-off you're accepting by fixing it?

**Candidate's answer** (asked for two hints; first pointed at First Normal Form and the earlier leading-wildcard reasoning, second asked directly whether one number per customer would let a normal index and `=` search work):
> "i think yeah i will held exactly 1 number per customer so when add the index to this i can search it more quickly but i will make the DB have the duplicate data"

**Feedback given:**
- Strong: landed on the right shape — one phone number per row instead of a delimited list, enabling indexing and equality search.
- Missing: the concept name — First Normal Form (1NF), a column's value must be atomic, not a delimited list.
- Missing: the concrete fix — a separate `customer_phone_numbers(id, customer_id FK, phone_number)` table, one row per number, not `phone_number_1`/`phone_number_2` columns.
- Missing/misconception: framed repeating `customer_id` as a foreign key across phone rows as "duplicate data" — that's just the normal, correct shape of a one-to-many relationship, not a cost. The real trade-off is needing a `JOIN` to get a customer's name and phones together, in exchange for correct indexing and unlimited phone counts.
- Missing: didn't restate that `LIKE '%...%'` with a leading wildcard can never use a B-tree index regardless of column shape, since the DB can't binary-search a pattern that could start anywhere in the string.

**Score: 4/10**

---

## Q3 — OOP, Design Patterns & Low Level Design

**Question:** A teammate writes this `@Service`:

```java
@Service
public class ReportGenerator {
    private int rowCount;

    public Report generate(List<Row> rows) {
        rowCount = rows.size();
        Report report = new Report();
        // ... some processing that reads rowCount a few lines later ...
        report.setSummary("Processed " + rowCount + " rows");
        return report;
    }
}
```

It passes all local testing (one request at a time). In production, under real concurrent traffic, support reports that some generated reports show a row count that clearly belongs to a *different* request entirely.

Given that `@Service` beans are singletons by default, why does this happen, and how would you actually fix it? Would switching the bean to prototype scope be a reasonable fix?

**Candidate's answer:**
> "because in the concurrent user it have many thread that access to the rowCount variable so that it skip the update we make before so we should use the atomic integer or synchronized the variable or lock this variable if we switch the prototype scope but we also use the service it will happen the bug"

**Feedback given:**
- Strong: correctly identified that concurrent requests hitting a shared mutable field on the same singleton instance is what causes one request's row count to leak into another's report.
- Missing: the proposed fixes (`AtomicInteger`, `synchronized`, locking) target the wrong problem — `rowCount` isn't shared state that needs safe combination across threads, it's per-request data that shouldn't live on a shared object at all. Even with perfect synchronization, Thread A's write can still be overwritten by Thread B's write before A reads it back — locking makes each read/write atomic but doesn't stop that interleaving.
- Missing: the actual fix — make `rowCount` a local variable inside `generate()`; local variables live on each thread's own stack frame, so there's nothing to race over. General principle: a singleton `@Service` should be stateless; anything that varies per call belongs in parameters/locals/return values, never an instance field.
- Missing: on prototype scope — the skepticism gestured at was directionally right but not landed: Spring typically resolves a bean's dependencies once, at the time the *injecting* bean is constructed, so a prototype-scoped `ReportGenerator` injected into a singleton controller would still, in practice, be reused as the same instance — prototype scope alone usually wouldn't fix this without a scoped proxy or `ObjectProvider`.

**Score: 4/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Spring Boot & JPA/Hibernate — `LazyInitializationException` from a lazy collection touched after the transaction/session closes (duplicate of 08-05's Q1 — pre-check missed it due to an older header format) | 4/10 |
| 2 | Databases: SQL, Indexing & Data Modeling — comma-separated phone numbers, 1NF violation, unsearchable `LIKE '%...%'` | 4/10 |
| 3 | OOP, Design Patterns & Low Level Design — mutable instance field on a singleton `@Service` leaking data across concurrent requests | 4/10 |

**Overall: 4.0/10**

**Strengths:**
- Reliable at diagnosing the *mechanism* once nudged toward it — session closing before lazy access, and shared state racing across threads — even when the terminology or full fix isn't there yet.
- Willing to reason out loud and revise mid-answer rather than guessing silently and stopping.

**Areas to review before the real interview:**
- **Read the question's constraint, not just the topic**: Q1 explicitly ruled out "just make it eager," and that's what got proposed anyway. A "not just X" in a question is a direct signal the expected answer is something else.
- **Synchronization tools solve races on genuinely shared values, not misplaced per-request state**: for Q3, the reflex to reach for `synchronized`/`AtomicInteger` on "concurrency + shared field" needs a sharper first question — should this even be a shared field?
- **1NF and "atomic values" as a named concept**: reached the right shape but needed two hints and still framed FK repetition across child rows as "duplicate data" rather than normal one-to-many modeling.
- **JPA session/transaction lifecycle reasoning needs to become automatic** — the "no `@Transactional` means the repository call's own transaction closes the instant it returns" chain explains a whole family of `LazyInitializationException` bugs, not just this one.
- **Process note for next session**: this session's opening question repeated 08-05's Spring Boot scenario almost exactly. Future sessions should cross-check topic novelty against `KNOWLEDGE_SUMMARY.md` directly (which already documents this exact LazyInitializationException mechanism) rather than only grepping raw session file headers, since older sessions use an inconsistent `## Question N —` heading format.
