# Mock Interview — Junior Backend Developer
**Date:** 2026-09-13

**Categories covered (3-question session, date-based seed since no timestamp was available: seed = 13×137 + 9×31 = 2060, start=1, step=7; sequence: Core Java & CS Fundamentals → Web Security & Frontend Basics → REST & API Design; opening category (1) differs from 09-07's opener (9) per the no-repeat-opener rule; all three sub-topics were checked against every prior session back to 07-04 to avoid reusing a scenario/mechanism already asked — Core Java's formula pick (Big-O of nested loops) had already been covered 07-08, so it shifted to exceptions (checked vs. unchecked, try-with-resources), untested across all 6 prior Core Java questions; Web Security's formula pick (calling a backend API from React) had been asked 3x (CORS twice, an out-of-order-response race once), so it kept the sub-topic but used the one untested mechanism inside it — `fetch()` not rejecting on 4xx/5xx; REST & API's formula pick (OpenAPI docs) had already been covered 08-12, so it shifted to DTOs vs. exposing JPA entities directly, untested across all 9 prior REST questions):** Core Java & CS Fundamentals, Web Security & Frontend Basics, REST & API Design

---

## Q1 — Core Java & CS Fundamentals

**Question:** A teammate's PR adds a startup step that reads an API key from a local file:

```java
public String loadApiKey(String path) {
    try {
        FileInputStream fis = new FileInputStream(path);
        byte[] data = fis.readAllBytes();
        return new String(data).trim();
    } catch (IOException e) {
        e.printStackTrace();
        return null;
    }
}
```

It works in their local testing. In code review you flag two things.

- `FileInputStream(path)` and `readAllBytes()` both declare `throws IOException` — is `IOException` a checked or unchecked exception, and how do you know just from looking at this method that it *had* to be wrapped in a try/catch (or declared with `throws`) to even compile?
- There's a resource-handling bug here independent of the exception handling — what is it, and how would `try-with-resources` fix it structurally (not just "it looks cleaner")?
- The `catch` block prints the stack trace and returns `null`, swallowing the error. Is that ever the right call here? Walk me through when you'd catch-and-handle locally versus let an exception propagate up — using this "API key missing at startup" scenario specifically.

**Candidate's answer** (needed two hints to move past "checked exceptions are ones the computer expects" to the actual class-hierarchy rule; admitted "I don't use try-with-resources at all" and needed the file-descriptor mechanism handed over directly after offering a GC/heap-memory misconception):
> "IOException i think it is checked exception right?" / "in my opinion i think the checked exception we can need to use the try catch when call it in the method but the unchecked is otherwise" / "i think the checked is this is the computer know i can have the exception so it mark it is checked but if it is the expcetion that from the user and the computer don't know when it happen it is unchecked this is what i think"
>
> "i think in the case if API key is missing we should not swallow it because it can make debugging more complicated and we can not know what it happen or miss" / "i think throw the exception so i will handle it and log the message to fix about this"
>
> "i don't use the try-with-resources at all" / "if the file keep openning when close the function i think the GC can not take it so the memory is more higher"

**Feedback given:**
- Strong: reached the right end conclusions — checked exceptions force a try/catch or throws, and propagate (don't swallow) so a missing API key fails app startup loudly instead of continuing with a `null` key that fails mysteriously downstream.
- Missing/misconception: checked vs. unchecked is a plain class-hierarchy rule, not "does the computer know it can happen" — checked = extends `Exception` but not `RuntimeException` (e.g. `IOException`); unchecked = extends `RuntimeException` or `Error` (e.g. `NullPointerException`).
- Missing/misconception: an unclosed `FileInputStream` is a **file-descriptor leak**, not a GC/heap-memory problem — GC doesn't track OS-level file handles. At scale this exhausts the process's file-descriptor limit (`IOException: Too many open files`), a different failure mode than `OutOfMemoryError`.
- Missing: `try-with-resources` guarantees `fis.close()` runs the instant the block exits (success or exception) — needed to be explained directly.

**Score: 3/10**

---

## Q2 — Web Security & Frontend Basics

**Question:** QA files a bug against your team's order-details page:

```jsx
function OrderDetails({ orderId }) {
  const [order, setOrder] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(`/api/orders/${orderId}`)
      .then(res => res.json())
      .then(data => setOrder(data))
      .catch(() => setError('Something went wrong loading this order'));
  }, [orderId]);

  if (error) return <ErrorBanner message={error} />;
  if (!order) return <Spinner />;
  return <OrderSummary order={order} />;
}
```

Report: "I opened a deleted order's URL directly. The backend logs show it returned `404` with `{ "error": "Order not found" }` — but the page just sat on the loading spinner forever. No error banner ever showed up."

- Why doesn't `.catch()` fire here, even though the server clearly responded with a `404`? What does `fetch()` actually treat as a failure?
- What would you change in this code to correctly detect and handle a `404` (or any non-2xx response)?
- Is this the same underlying mechanism as a CORS failure, or a genuinely different one? How would the symptoms differ if this had been a CORS error instead?

**Candidate's answer** (initially misdiagnosed this as a backend soft-delete design issue; needed the `res.json()` mechanism explained directly, then correctly traced that `setOrder` would receive the error payload; asked for the final answer outright rather than completing the fix and CORS-comparison independently):
> "first i think the handle in backend don't delete the order it just change the status of the order, so that the .catch() is never fire and make the app spin forever... i think the code above is good enough the thing need to change is in backend"
>
> "oh so when use the res.json() it mark it successful and the order is set as a data but the order have the value right now it can make the app spin forever"
>
> "i will check the res object if it return the status code in 2xx i will handle this but if is in the range 4xx i will return false" / "can you answer this"

**Feedback given:**
- Strong: once redirected away from the backend misdiagnosis, correctly traced that `res.json()` parses successfully regardless of HTTP status, and that this — not a backend design gap — is why `.catch()` never fires.
- Missing/corrected by interviewer: the actual crash-site fix requires `throw`ing inside the first `.then()` when `!res.ok`, which rejects the promise chain and routes it to `.catch()` — "return false" doesn't do anything in a promise chain feeding React state, and this had to be handed over.
- Missing/corrected by interviewer: a CORS block rejects `fetch()`'s promise at the network level (`TypeError`, no readable `Response` at all) before any `.then()` runs — a fundamentally different failure point than a real, fully-readable `404` response, which `fetch()` treats as a completed request.
- Also corrected: the interviewer's own scenario framing ("stuck on spinner forever") was imprecise — with `setOrder` receiving the error payload, `order` becomes truthy and the component would render `<OrderSummary>` with garbage data rather than spin indefinitely; the underlying `.catch()`-never-fires mechanism the question tested was unaffected by this correction.
- Asked for the answer outright on both the fix and the CORS comparison rather than continuing to reason through them.

**Score: 3/10**

---

## Q3 — REST & API Design

**Question:** A teammate opens a PR for `GET /users/{id}`:

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
}

@Entity
public class User {
    private Long id;
    private String email;
    private String passwordHash;

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
}
```

It works fine in Postman when they test it.

- Name two concrete problems with returning this `User` entity directly from the controller — be specific to *this* entity, not just "it's bad practice."
- What would you change instead?
- Six months later, product wants the response to include a computed `orderCount` field, without adding any new column to the `users` table. How does the fix from part 2 make that change easier than it would be if the controller kept returning the entity directly?

**Candidate's answer** (correctly proposed the DTO + JOIN FETCH fix unprompted, but never named the entity's most concrete flaw and mislabeled the lazy-loading exception):
> "i think i can have the problem is return the entity and the N+1 query in that and when use call the variable in this it can have the exception is Close Transaction if i right. so the change is think i will create the DTO and i will write a query that JOIN 2 table User, Order and get all at once so that and i will use the DTO to contains it the infomation of User and the List<Order> of user so when after sixmonth when the response must to have a order count we just user.getOrders()[id].getOrderCount() so that it can avoid 2 problem and make our DB keep running not slow or have any error"

**Feedback given:**
- Strong: unprompted, correctly proposed the real working fix — a DTO backed by a `JOIN FETCH` query — matching one of the two standard fixes for this class of bug.
- Strong: right general area on the lazy-loading issue, even though the exact exception name and mechanism were off.
- Missing: never named the sharpest, most entity-specific problem — `passwordHash` sits directly on `User` and returning the entity means every `GET /users/{id}` call leaks the password hash to any client. This is the answer the question was specifically fishing for with "be specific to *this* entity."
- Missing/misconception: this isn't an "N+1 query" (that's from looping over *many* parent entities, each triggering its own lazy-load) — it's a single `LazyInitializationException`, thrown because `findById`'s session/transaction closes the moment the repository call returns, and Jackson later tries to serialize the still-lazy `orders` collection outside any open session.
- Missing: the DTO fix should explicitly exclude `passwordHash` from the DTO shape entirely — this was implied but never stated.
- Missing: the part-3 answer (`user.getOrders()[id].getOrderCount()`) doesn't correspond to a real computation — the actual point is that `orderCount` becomes a plain field on the DTO (e.g. `orders.size()`, or a dedicated `COUNT` query) added with zero schema change, precisely because the DTO is a view-shape under your control — bolting a computed field onto the `@Entity` itself would instead mix persistence mapping with response-shaping and hit the same lazy-loading problem from part 1.

**Score: 4/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Core Java & CS Fundamentals — checked/unchecked exceptions & resource leak | 3/10 |
| 2 | Web Security & Frontend Basics — `fetch()` not rejecting on non-2xx vs. CORS | 3/10 |
| 3 | REST & API Design — DTO vs. exposing a JPA entity directly | 4/10 |

**Overall: 3.3/10**

**Strengths:**
- Consistently lands on the right high-level conclusion (propagate the exception, use a DTO + JOIN FETCH, distinguish a real `404` from a CORS block) even when the underlying mechanism is initially wrong or missing entirely.
- Comfortable saying "I don't know" and asking for hints rather than bluffing through a guess — a healthy interview habit, even though it lowered the hint-adjusted score this session.
- Traces through code step by step once a key fact is supplied (e.g. correctly walked through the `.then()` chain after being told what `res.json()` does with a 404 body).

**Areas to review before the real interview:**
- Checked vs. unchecked exceptions is a class-hierarchy fact (extends `Exception`-not-`RuntimeException` vs. extends `RuntimeException`/`Error`), not a "does the computer expect it" heuristic — worth memorizing outright.
- Unclosed streams/connections leak OS-level file descriptors, not JVM heap memory — don't reach for "GC"/"memory" as the explanation for a resource leak.
- `fetch()`'s promise only rejects on network-level failures; any HTTP status (including 4xx/5xx) is still a "successful" fetch as far as the promise chain is concerned — always check `res.ok`/`res.status` explicitly, and know that `throw`ing inside a `.then()` is what redirects a chain into `.catch()`.
- When asked "what's wrong with exposing this entity," scan the entity's own fields for something concretely dangerous (like a `passwordHash`) before reaching for general mechanisms — that specific angle was missed entirely this session in favor of a more generic (and mislabeled) answer.
- Get precise with exception names: this session conflated "N+1 query" (a looping, many-queries bug) with a single-entity `LazyInitializationException` (a closed-session bug) — different root causes, different fixes.
