# Mock Interview — Junior Backend Developer
**Date:** 2026-07-08

## Categories this session
Seed = 8×137 + 7×31 = 1313 → starting_index = 4, step = 9
1. Spring Boot & JPA/Hibernate
2. Databases: SQL, Indexing & Data Modeling
3. OOP, Design Patterns & LLD
4. Core Java & CS Fundamentals
5. Tooling: Git, Build & Environment

---

## Q1 — Spring Boot & JPA/Hibernate

**Question:** You're reviewing a teammate's PR. Their `@Service` class has:
```java
@Autowired
private OrderRepository orderRepository;
```
with no constructor at all. You leave a comment asking them to switch to constructor injection instead. They push back: "It works fine either way, why does it matter?" What do you tell them — what's actually different between field injection and constructor injection here?

**Candidate answer (after 1 hint):** "i think if we use the constructor approach we can manage the lifecycle of the field, and easily to test"

**Score: 5/10**

**Strong:**
- Testability point is right — constructor injection is the correct instinct to reach for.

**Missing:**
- "Manage the lifecycle" isn't quite the right frame — Spring treats the bean as a singleton either way. What changes: constructor injection lets the field be `final`, guaranteeing it's set the instant the object exists (no null window); field injection builds the bare object first, then reflects the field in afterward.
- Fail-fast: constructor injection fails app startup immediately with a clear error if the bean is missing; field injection can fail more confusingly.
- Testing mechanism: constructor injection allows `new OrderService(mockRepo)` in a plain JUnit test with zero Spring involvement.
- Bonus: constructor injection surfaces "too many dependencies" (SRP smell) at a glance; field injection hides it.

**Tip:** Anchor answers on a concrete failure mode ("what breaks, and when") instead of a general word like "lifecycle."

---

## Q2 — Databases: SQL, Indexing & Data Modeling

**Question:** You design:
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  email VARCHAR(255),
  name VARCHAR(255)
);
```
A month later, users end up with duplicate accounts sharing the same email. Didn't `PRIMARY KEY` already protect against duplicates? What's missing, and what's the difference between what `PRIMARY KEY` protects and what you'd need here?

**Candidate answer:** "i think the id is just different , it can not protect the duplicate field , it just identify that the id is different it can not identify the duplicate name on other field we should add the UNIQUE to the field that can duplicate, and using in the business logic"

**Score: 7/10**

**Strong:**
- Correctly identified that `PRIMARY KEY` only guarantees uniqueness on `id`, not `email`.
- Correct fix: add a `UNIQUE` constraint on `email`.

**Missing:**
- "Business logic" check alone isn't sufficient: two concurrent requests can both check "does this email exist?", both see no match, and both insert — a race condition producing the exact bug reported. The DB-level `UNIQUE` constraint is what actually closes this, since it's enforced atomically regardless of caller. App-level checks are useful for a friendly error message, but the constraint is the real guarantee.

**Tip:** When reasoning about "is this a real guarantee," ask "what happens if two requests hit this at the same millisecond?"

---

## Q3 — OOP, Design Patterns & LLD

**Question:** A teammate builds a form validation (2 static rules — email format, password length, unchanged for a year) as a `ValidationStrategy` interface + 5 implementing classes + a factory. You think it's overkill. How would you push back in review, and what's a pattern already baked into a framework you use (e.g. Spring) that "earns its keep," in contrast to hand-rolling one here?

**Candidate answer (after 1 hint):** "i think it are too much code just for the ValidationStrategy, in the example use the Strategy Pattern and FactoryPattern, it good, but it is not essential in this case; i think in Spring we have use some annotation like @NotNull, @Length"

**Score: 6/10**

**Strong:**
- Correct instinct that the pattern is disproportionate for static, unchanging rules.
- Good framework example: Bean Validation annotations (`@NotNull`, `@Size`/`@Length`) — worth noting these are backed by `ConstraintValidator` implementations, i.e. Spring already uses Strategy here without exposing the plumbing.

**Missing:**
- The "why" was thin — needed the YAGNI framing explicitly: patterns like Strategy pay off when there's a real variation point (rules added often, or swapped at runtime). No evidence of that here.
- A concrete review comment: "Let's use `@NotNull`/`@Size` (or plain `if` checks) for now; revisit Strategy if we get a 3rd/4th rule that needs independent swapping."

**Tip:** When judging if a pattern is justified, ask "what specifically is going to vary, and how often?" If you can't name it, it's premature.

---

## Q4 — Core Java & CS Fundamentals

**Question:** A report endpoint loops 10,000 `orders` × 5,000 `customers` to find matches by `id` — takes 30 seconds. Why is this slow in Big-O terms, and how would you fix it?

**Candidate answer:** "the loop is about 10000 x 5000 = 50000000; O(5.10^7) is very large complexity... i will fix it by create the set for each id of users (loop 5000 customers), use the second loop for order and check if the id is exist in set i will build the report; it will take O(5000) + O(10000) so under 1 second"

**Score: 8/10**

**Strong:**
- Correct Big-O reasoning and correct quantification connecting it to the 30s runtime.
- Correct fix direction: precompute a hash lookup, then single pass — O(n + m).

**Missing:**
- A `Set<CustomerId>` only tells you existence, not the actual customer needed to build the report line — should be `Map<CustomerId, Customer>` built once, then `map.get(...)` for O(1) retrieval of the full object.
- Minor precision: it's O(n × m), not O(n²), since the two lists have different sizes.

**Tip:** When reaching for a hash structure, ask "do I need to know if it exists, or do I need the object?" — that decides `Set` vs `Map`.

---

## Q5 — Tooling: Git, Build & Environment

**Question:** Adding a new dependency causes `NoSuchMethodError` on a Jackson method at runtime, though nothing in your own code changed. You suspect a version conflict from transitive dependencies. How would you confirm it, and how would you fix it?

**Candidate answer (after 1 hint):** "yeah i will see the maven dependency and check if it exist if it don't i will go to the stackoverflow to find the solution"

**Score: 3/10**

**Strong:**
- Directionally right instinct — inspect dependency state before assuming code changes.

**Missing:**
- The specific command: `mvn dependency:tree` prints the resolved graph and flags dropped versions as "(conflict)". Gradle equivalent: `./gradlew dependencies` or `dependencyInsight --dependency <name>`.
- Resolution rule: Maven uses **nearest-wins** — the declaration closest to your project wins regardless of which version is newer.
- The fix: explicitly pin the version in your own `pom.xml` (direct declarations win), use `<dependencyManagement>` in multi-module projects, or `<exclude>` the unwanted transitive version.

**Tip:** "It's fine to say what you'd look up" is true and rewarded — but name *what* you'd search for ("Maven nearest-wins dependency conflict resolution") rather than "Stack Overflow" generically.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Spring Boot & JPA/Hibernate — constructor vs. field injection | 5/10 |
| 2 | Databases — PRIMARY KEY vs. UNIQUE constraint | 7/10 |
| 3 | OOP/Design Patterns & LLD — recognizing over-engineering | 6/10 |
| 4 | Core Java & CS Fundamentals — Big-O of nested loops | 8/10 |
| 5 | Tooling: Git, Build & Environment — Maven dependency conflicts | 3/10 |

**Overall: 5.8/10**

**Strengths:**
- Strong algorithmic instincts — correctly quantified the nested-loop cost and reached for a hash-based fix (Q4, best answer).
- Solid grasp of core DB semantics — clearly separated what `PRIMARY KEY` guarantees from what `UNIQUE` guarantees (Q2).
- Good gut check for over-engineering — correctly flagged the Strategy/Factory setup as disproportionate for two static rules (Q3), even though the reasoning needed a hint.

**Areas to review before the real interview:**
- Build tools (Maven/Gradle): missing the diagnostic command (`mvn dependency:tree`) and the resolution rule (nearest-wins) — a common real debugging scenario, worth drilling.
- Explaining the "why," not just the "what": in Q1 and Q3, the conclusion was right but needed a hint to reach the underlying mechanism (immutability/fail-fast; YAGNI/variation point).
- Map vs. Set precision: in Q4, reached for a hash structure correctly but conflated "check existence" (`Set`) with "retrieve the object" (`Map`) — a common junior mixup worth locking in.
