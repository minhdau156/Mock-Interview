# Mock Interview — Junior Backend Developer
**Date:** 2026-08-09

**Categories covered (3-question session, seed-random start (seed=1481, start=2, step=1), categories 5/6/9 skipped because they repeat 2026-08-08's topics — REST & API Design, Concurrency & Locking, System Design — sequence otherwise: OOP/LLD → Databases → Spring/JPA):** OOP/Design Patterns & LLD, Databases: SQL/Indexing/Data Modeling, Spring Boot & JPA/Hibernate

---

## Q1 — OOP, Design Patterns & LLD

**Question:** In a code review, you see this in `OrderService`:

```java
public class OrderService {
    public void checkout(Order order) {
        PaymentGateway gateway = new StripePaymentGateway();
        gateway.charge(order.getTotal());
        orderRepository.save(order);
    }
}
```

A teammate comments: "This violates dependency inversion, and it's also going to make your life miserable in a few weeks." They don't say anything more.

What do you think they mean — what's actually wrong with this, and how would you change it?

**Candidate's answer:**
> "your teammate is right / you violate the dependency inversion because you initial the PaymentGateway in the service so it will tight couple together so when the StripePaymentGateway have the change in the constructor we need to fix it in OrderService so it is not suitable for us and it hard for testing we hard to mock the Object in the function and it will call the real PaymentGateway so it is avoidable / so we need to change it, we should create the PaymentGateway as a variable of OrderService and use the Constructor Dependency or Setter Dependency for init the GateWay / it will loose couple 2 service and we are more easy to testing we can Mock it easily and if the PaymentGateway have change the code of constructor we don't need to change or fix in OrderService"

**Feedback given:**
- Strong: correctly named the violation (DIP) and connected it to tight coupling; correctly identified the testability problem (can't mock, hits the real gateway); correctly proposed constructor injection of the abstraction as the fix.
- Missing: the formal DIP statement — both high- and low-level modules should depend on abstractions. The subtlety here is the abstraction (`PaymentGateway`) already exists as a type; the violation is that `OrderService` ignores it and instantiates the concrete class anyway.
- Missing: framed the consequence too narrowly ("if Stripe's constructor changes we need to fix OrderService"); the bigger risk is never being able to swap providers or unit-test checkout logic without hitting the real API.
- Missing: the Spring-specific shape of the fix — `StripePaymentGateway implements PaymentGateway` as `@Service`, `OrderService` takes `PaymentGateway` via constructor, Spring wires the concrete bean automatically.

**Score: 7/10**

---

## Q2 — Databases: SQL, Indexing & Data Modeling

**Question:** Your team's `orders` table has grown to 20 million rows. A support engineer reports that this query, which used to be instant, now takes 4 seconds:

```sql
SELECT * FROM orders WHERE customer_id = 42 AND YEAR(created_at) = 2026;
```

There's already an index on `(customer_id, created_at)`. A teammate says "the index should be handling this — why is it slow?"

What's actually going on here, and what would you change?

**Candidate asked two clarifying questions** — the data format of `created_at` (a standard `TIMESTAMP`), and how to compare two timestamps in SQL (standard operators / `BETWEEN`) — before answering.

**Candidate's answer:**
> "because we use the the composite index so the first condition 'customer_id = 42' is satifie so it will have the Index Scan but when goes to the second condition it is 'YEAR(created_at) = 2026' it sort the data of created_at is TIMESTAMP so when use the YEAR() function is like the char so it can not match and it need to full scan the table to find some column that YEAR(created_at) = 2026 of customer have id is 42 in the table have 20 million row it will take much time / so if we want to fix we should compare the created_at with TIMESTAMP / i will re-write the query like this: SELECT * FROM orders WHERE customer_id = 42 AND created_at >= '2026-01-01 00:00:00' AND created_at < '2027-01-01 00:00:00';"

**Feedback given:**
- Strong: correctly applied the leftmost-prefix rule (customer_id is the leading index column); correctly transferred the "function on an indexed column defeats the index" principle from the already-known `UPPER(email)` case to a new context (`YEAR()` on a `TIMESTAMP`); correct fix (direct range comparison).
- Missing: the general mechanism stated precisely — a function wrapped around an indexed column forces per-row computation before comparison, since the index is built on raw stored values, not the function's output.
- Missing: overstated the cost as a full scan of the whole table — leftmost-prefix still bounds the scan to just that customer's rows; the function only defeats the index for the second predicate, not the first.

**Score: 6/10**

---

## Q3 — Spring Boot & JPA/Hibernate

**Question:** In a PR, a teammate adds this entity:

```java
@Entity
@Data
public class Customer {
    @Id
    @GeneratedValue
    private Long id;
    private String email;
    private String name;
}
```

(`@Data` is Lombok — it generates getters, setters, `equals()`, `hashCode()`, and `toString()` for all fields.)

What would you flag in review here, and why?

**Candidate needed two hints** (how Hibernate actually instantiates an entity; whether `@Data` alone generates a no-args constructor) **and asked one clarifying question** (compile-time vs. runtime failure — confirmed runtime) before answering.

**Candidate's answer:**
> "i think the hibernate will create the new Customer using no argument constructor and set the variable using setter function so when just use the @Data it will have the error like you mentioned so we need to add the AllArgsConstructor and NoArgsCOnstructor to it"

**Feedback given:**
- Strong: correctly described the real JPA/Hibernate mechanism — entities are instantiated via reflection using a no-args constructor, then populated, not via an all-args constructor.
- Missing: in this exact snippet, that risk doesn't actually fire — `@Data` implicitly includes `@RequiredArgsConstructor`, which generates a zero-arg constructor here since there are no `final`/`@NonNull` fields. The proposed fix doesn't address an actual bug in this code (it only matters once a `final`/`@NonNull` field is added later).
- Missing: the real defect — `@Data` generates `equals()`/`hashCode()` over **all** fields, including the mutable, JPA-managed `id`. A transient `Customer` (`id == null`) placed in a `HashSet`/used as a `HashMap` key before persisting will have its hashCode change the instant Hibernate assigns a real `id` on save — the same mutable-hashCode-key bug already learned in Core Java (`RequestKey`, 08-03), now applied to an entity's transient→persistent transition.
- Missing: the fix — avoid blanket `@Data` on `@Entity`; use `@Getter`/`@Setter` plus `@EqualsAndHashCode(of = "id")` (or hand-written equals/hashCode on `id` with a null-safe check). Secondary risk: `@Data`'s generated `toString()`/`equals()` can cause infinite recursion if a bidirectional relationship is added later.

**Score: 4/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | OOP/LLD — Dependency Inversion, constructor-injected abstraction vs. concrete instantiation | 7/10 |
| 2 | Databases — function-on-indexed-column (`YEAR()`) defeating a composite index | 6/10 |
| 3 | Spring/JPA — `@Data` on an `@Entity`, no-args constructor vs. the real equals/hashCode risk | 4/10 |

**Overall: 5.7/10**

**Strengths:**
- Diagnosed the DIP violation unprompted and proposed the standard constructor-injection fix, tying it to concrete coupling/testability consequences.
- Transferred an already-learned principle (function-on-indexed-column defeats index, from `UPPER(email)`) to a new context (`YEAR()` on a `TIMESTAMP`) with only one clarifying nudge.
- Correctly reasoned about JPA's reflection-based instantiation mechanism, even though it wasn't the actual bug in this specific snippet.

**Areas to review before the real interview:**
- DIP (Q1): state the formal principle explicitly (both high- and low-level modules depend on abstractions), and lead with swappability/pure-unit-testing as the payoff, not just "avoiding a fix when the constructor changes."
- Index-defeating functions (Q2): get to range-comparison syntax without a nudge, and be precise that leftmost-prefix still bounds the scan to one customer's rows rather than the full table.
- `@Data` on JPA entities (Q3): review the equals/hashCode-over-mutable-fields gotcha — it's the same mutable-key bug already learned in Core Java, now applied to entities transitioning from transient to persistent.
