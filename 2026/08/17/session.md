# Mock Interview — Junior Backend Developer
**Date:** 2026-08-17

**Categories covered (3-question session, seed-random start (seed=2577, start=8, step=3), sequence: Web Security & Frontend Basics → Core Java & CS Fundamentals → Spring Boot & JPA/Hibernate; opening category (8) differs from 2026-08-12's opener (9) per the no-repeat-opener rule; Q2's sub-topic was swapped from ArrayList vs LinkedList to String immutability at the candidate's request, same category):** Web Security & Frontend Basics, Core Java & CS Fundamentals, Spring Boot & JPA/Hibernate

---

## Q1 — Web Security & Frontend Basics

**Question:** A teammate's React component keeps hammering the network tab with the same request over and over, and the tab starts to freeze:

```jsx
function ProfilePage({ userId }) {
  const [profile, setProfile] = useState(null);
  const options = { include: 'address' };

  useEffect(() => {
    fetchProfile(userId, options).then(setProfile);
  }, [options]);

  if (!profile) return <Spinner />;
  return <ProfileCard profile={profile} />;
}
```

They ask you: "I put `options` in the dependency array like the linter told me to — why is it still looping forever?" What's actually going on here, and how would you fix it?

**Candidate's answer:**
> "i remember that the useEffect if have the dependency it will rerender again if the dependency change but i think the problem is the react think when fetch profile it will make the file re render so and the option is create again and so that react think it is change the options and it loop the useEffect forever / i will change it use the useState so it the react will not think it change when render it just change when we use the setOptions"

**Feedback given:**
- Strong: correctly spotted that `options` is a new object created on every render — that's the actual root cause.
- Strong: fix direction is sound — keeping the same object reference across renders instead of recreating it breaks the loop.
- Missing: the precise mechanism name — `useEffect`'s dependency comparison uses reference/identity equality, not deep/value equality. `{ include: 'address' }` looks identical every render, but it's a different object in memory each time.
- Missing: the full causality chain (candidate had the pieces reversed) — render → `options` created fresh → effect sees a new reference → runs `fetchProfile` → `.then(setProfile)` updates state → triggers a re-render → `options` created fresh again → loop.
- Missing: `useState` technically works (the initial object is only constructed once, reference stays stable unless `setOptions` is called), but it's not the idiomatic tool since `useState` is meant for values that change over time. `useMemo(() => ({ include: 'address' }), [])` or hoisting the literal outside the component is the standard fix.

**Score: 6/10**

---

## Q2 — Core Java & CS Fundamentals

**Question:** A junior teammate wrote this to build a CSV export:

```java
public String buildCsv(List<Order> orders) {
    String csv = "";
    for (Order o : orders) {
        csv += o.getId() + "," + o.getTotal() + "\n";
    }
    return csv;
}
```

It works fine in tests with 20 rows, but in production with 50,000 orders, this single method call is eating several seconds of CPU time and shows up as the top hot spot in a profiler. Why does something this "simple" get slow at scale, and what would you change?

**Candidate's answer:**
> "because the String is the immutable object so when we use the operator + it will create another object and attach the so when if have 50000 orders it will make the memory is up and it will down the CPU / so the solution we should use StringBuilder the mutable object it will use current object to change which we want"

**Feedback given:**
- Strong: nailed the root cause — String immutability means `+=` doesn't mutate the existing string, it allocates a brand-new object every time.
- Strong: correct fix — `StringBuilder`, which mutates an internal buffer instead of allocating a new object per operation.
- Missing: the specific complexity jump — it's O(n²) total work, not just "more memory." Iteration *i* copies all characters accumulated so far into the new, bigger string, so total characters copied across 50,000 iterations grows quadratically.
- Missing: what `csv += ...` compiles to — roughly `csv = new StringBuilder(csv).append(...).toString()` — a brand-new `StringBuilder` created and the entire existing string copied into it, every single iteration, then converted back to an immutable `String`.
- Missing: the complete fix is declaring one `StringBuilder` outside the loop and appending to the same instance every iteration, then `.toString()` once at the end — that's what actually gets to O(n).

**Score: 7/10**

---

## Q3 — Spring Boot & JPA/Hibernate

**Question:** Reviewing a PR for `Order`/`OrderItem`:

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}
```

Later, in a completely unrelated admin reporting endpoint, someone writes code that loads an `Order`, builds a temporary "items to display" list by filtering `order.getItems()`, and — just to keep the UI list clean — calls `order.getItems().remove(item)` on an item that shouldn't show in that particular report view. No explicit delete is called anywhere.

A support ticket comes in: that order item is now actually gone from the database. What's happening here, and what about this mapping made that possible?

**Candidate's answer:**
> "i think when the hibernate load the OrderItem it will keep it to items variable but in this code it call the new ArrayList<>() so it will make the items will empty / and the second the UI that render is temporary so we don't know does it have the data it the items or not so when we delete the OrderItem it keep in the DB / the solution is we need to make change is delete the new ArrayList keep the data jpa and hibernate get the value to the variable and we should check the data before testing any thing"

**Feedback given:**
- Missing/misconception: the field initializer `= new ArrayList<>()` only matters for a brand-new, transient `Order` built with `new Order()`. When Hibernate loads an existing row, it replaces that field with its own managed collection wrapper (`PersistentBag`) during hydration — the initializer is irrelevant there and doesn't empty anything.
- Missing: the real mechanism is `cascade = CascadeType.ALL` + `orphanRemoval = true`. `orphanRemoval = true` tells Hibernate "removed from this collection = delete it." `order.getItems().remove(item)` mutates the real, managed collection directly (no copy involved in the code shown) — Hibernate's dirty-checking sees the change at flush time and issues a real `DELETE`.
- Missing: the intent mismatch that makes this dangerous — the report author thought they were building a filtered display list, but mutating the entity's live collection instead of a copy means Hibernate can't distinguish "filtering for display" from "business logic removed this."
- Missing: correct fix — never call mutating methods on an entity's live collection in read-only code; copy it first (`order.getItems().stream().filter(...).toList()`). `orphanRemoval = true` should be a deliberate choice, not a default alongside `cascade = ALL`.

**Score: 3/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Web Security & Frontend Basics — `useEffect` dependency identity | 6/10 |
| 2 | Core Java & CS Fundamentals — String immutability / `StringBuilder` | 7/10 |
| 3 | Spring Boot & JPA/Hibernate — cascade + `orphanRemoval` | 3/10 |

**Overall: 5.3/10**

**Strengths:**
- Consistently landed on the right symptom and often the right fix direction without needing a hint (object recreated → loop; immutable String → StringBuilder).
- Willing to reason through unfamiliar mechanics out loud rather than guessing blindly.

**Areas to review before the real interview:**
- JPA cascade/`orphanRemoval` semantics, and the broader point that mutating a managed entity collection inside an open transaction is a real write — today's weakest spot.
- Naming the precise *why*, not just the *what*: reference-equality behind `useEffect` deps, quadratic copying behind string concatenation, dirty-checking behind cascade deletes.
- Entity lifecycle: what Hibernate does when hydrating an existing row vs. constructing a new object with `new` — the field-initializer mix-up in Q3 suggests this boundary needs review.
