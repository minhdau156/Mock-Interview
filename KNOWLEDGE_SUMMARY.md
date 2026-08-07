# Mock Interview — Knowledge Summary

**Covers sessions:** 2026-07-04, 07-05, 07-06, 07-08, 07-10, 07-14, 07-16, 07-29, 08-01, 08-02, 08-03, 08-04, 08-05, 08-06, 08-07 (15 sessions, 65 questions)
**Role target:** Junior Backend Developer (Java/Spring Boot)

This is a study reference, not just a log. Each category below gives the full mechanism — not only what was missed in a session, but the concept in enough depth to explain it cold in the real interview, with code where it clarifies the point.

---

## 1. Score Trend

| Date | Overall | Categories covered |
|------|---------|---------------------|
| 07-04 | 4.0/10 | Concurrency, Testing/Debugging, Web/Frontend, System Design, Git Tooling |
| 07-05 | 4.2/10 | Databases, Git Tooling, Testing/Debugging, Spring/JPA, Core Java |
| 07-06 | 6.6/10 | Git Tooling, System Design, Web/Frontend, Testing/Debugging, Concurrency |
| 07-08 | 5.8/10 | Spring/JPA, Databases, OOP/LLD, Core Java, Git Tooling |
| 07-10 | 5.0/10 | Web/Frontend, REST API, OOP/LLD, System Design, Concurrency |
| 07-14 | 6.8/10 | Concurrency, System Design, OOP/LLD, REST API, Web/Frontend |
| 07-16 | 5.2/10 | Git Tooling, Core Java, OOP/LLD, Databases, Spring/JPA |
| 07-29 | 4.6/10 | Core Java, Git Tooling, System Design, Web/Frontend, Testing/Debugging |
| 08-01 | 6.0/10 | Concurrency, Databases, Git Tooling, Testing/Debugging, Spring/JPA |
| 08-02 | 4.2/10 | Databases, Spring/JPA, REST API, Concurrency, Testing/Debugging |
| 08-03 | 5.3/10 | REST API, OOP/LLD, Core Java (3-question session, categories hand-picked for least-tested coverage) |
| 08-04 | 5.3/10 | Testing/Debugging, Concurrency, REST API (3-question session, seed-random categories, but sub-topics deliberately steered away from already-drilled angles toward untested ones) |
| 08-05 | 5.7/10 | Spring/JPA, Testing/Debugging, Git Tooling (3-question session, seed-random categories, sub-topics steered to fresh angles: `LazyInitializationException`, testing pyramid, merge vs. rebase) |
| 08-06 | 5.3/10 | Core Java, System Design, Web Security/Frontend (3-question session, seed-random categories with all three of 08-05's categories explicitly excluded per request, sub-topics steered to fresh angles: Integer caching, stateless multi-instance sessions, cookie tampering vs. JWT signing) |
| 08-07 | 4.0/10 | Git Tooling, OOP/LLD, Databases (3-question session, seed-random categories (seed=1207, start=8, step=1) with categories 8/9/1 skipped because they repeated 08-06's — Web Security, System Design, Core Java — sub-topics steered to fresh angles: responding to PR review feedback with evidence, tell-don't-ask/anemic domain model, soft delete vs hard delete) |

Trend is upward but noisy (4.0 → 6.8 with dips) — scores depend heavily on which specific angle a topic is tested from, not just topic familiarity. Best sessions (07-06, 07-14) both landed a clean 8/10 on **concurrency/race-condition tracing** and handled **CORS/system design** reasonably. Weakest sessions (07-04, 07-05) struggled most with **debugging methodology** and **request-lifecycle/exception fundamentals**. 07-29 was the first session to deliberately target sub-topics *not yet documented here* rather than seed-randomized picks — the dip to 4.6 reflects fresh, previously-untested gaps (merge conflict mechanics, mocking anti-patterns) surfacing for the first time, not regression on known material. 08-01 continued that deliberate-gap-targeting approach and landed 6.0/10 — clean, correct-on-first-try answers on composite-index reasoning and N+1 recognition, but exposed a fresh and fairly deep gap in test-driving a bug fix, plus a partial gap on why in-process locks don't coordinate across app instances. 08-02 kept the seed-random category picker but required every question's *content* to differ from anything already documented here — the dip to 4.2 reflects several previously-untested mechanisms landing at once: self-invocation bypassing `@Transactional`'s proxy, mistaking lock-order-inversion deadlocks for simple contention/timeout, and a non-timing (stack-depth threshold) intermittent bug that didn't fit the timing-bug debugging framework already learned. The one strong showing (7/10, OFFSET-vs-cursor pagination) shows the "reason from first principles about a new mechanism" skill transfers when the scenario doesn't collide with an already-memorized-but-wrong rule. 08-03 broke from the seed algorithm entirely — categories were hand-picked for lowest historical coverage (REST API, OOP/LLD, Core Java) and cut short to 3 questions by request. The 5.3 average was pulled down by a genuine LLD process gap (state modeling skipped, plus a Factory/inventory mislabeling repeat) rather than a misapplied Spring/DB rule — meanwhile the REST API answer (7/10) was the sharpest single answer of the session, correctly choosing `409` over the common junior default of `400` for a state-conflict error with no hint needed. 08-04 kept the seed-random category picker (Testing & Debugging, Concurrency & Locking, REST & API Design) but deliberately steered each question's sub-topic away from angles already drilled to 8/10 in prior sessions (the counter race-condition, the timing-bug debugging framework) toward genuinely untested ground — the 5.3 average was pulled down by a flat non-attempt on the flaky-test question (extensive clarifying questions, but no independent answer even after a hint pointing at the weak assertion and multi-behavior test), while the REST API answer (8/10) was again the strongest single answer of the session, correctly reversing a nested-vs-flat route decision the moment the query's scope changed from one customer to all customers, with no hint needed. 08-05 continued the seed-random category picker with sub-topics steered to genuinely fresh ground (Spring/JPA, Testing & Debugging, Git Tooling) — the 5.7 average was pulled down by the weakest single answer of the session (4/10, merge vs. rebase), where basic mechanics had to be explained from scratch and a fresh misconception surfaced (believing rebase preserves "who did it" better than merge, when both preserve authorship identically); the strongest answer (7/10, testing-pyramid restructuring) showed a genuine mid-answer self-correction — refining an initially-imprecise rule ("use unit tests when there's a lot of logic") to the sharper "DB/API call in the test means integration test" without being prompted to. 08-06 excluded all three of 08-05's categories per explicit request, landing on Core Java, System Design, and Web Security/Frontend instead — the 5.3 average was pulled down by a flat non-independent-answer on Integer caching (needed two hints and still couldn't name `Integer.valueOf`'s -128..127 cache range even after being told the two print statements behave differently), while the System Design answer (7/10, stateless multi-instance sessions) was the strongest of the session — correctly deriving, after only one nudge, that an in-memory session `Map` isolated per server instance causes random logouts once a second instance joins a round-robin load balancer, and proposing Redis externalization unprompted; the Web Security answer (6/10) correctly disagreed with storing role data in a plain cookie and named JWT, but framed the risk as "a hacker reading it" (confidentiality) rather than the more direct issue tested — the user tampering with their own cookie to self-escalate (integrity) — and didn't name signature verification as the actual mechanism that stops it. 08-07 broke from the seed-random category picker's default run when three of its five natural picks (Web Security, System Design, Core Java) collided with 08-06's topics, landing instead on Git Tooling, OOP/LLD, and Databases — the 4.0 average, the lowest since 07-05, was pulled down primarily by a genuine non-attempt on the Git Tooling question (2/10: asked for two hints on how to respond to a reviewer's disagreement, then gave no independent answer at all, worse than the usual "needed a hint but eventually reasoned it out" pattern seen elsewhere); the OOP answer (4/10) showed a real, hint-free SRP instinct but restructured logic within the wrong class entirely, missing the tell-don't-ask/anemic-domain-model principle the question targeted; the strongest answer of the session (6/10, soft delete vs. hard delete) correctly chose the right mechanism unprompted but missed the sharper trade-offs (query-filtering burden, restore-data-integrity) in favor of a table-growth argument that overstates the actual performance risk.

---

## 2. Critical Misconceptions (answered confidently, but wrong — highest priority)

These aren't just gaps — they were stated as fact and are incorrect. Fix these first.

- **`@Transactional` does not coordinate across separate requests/transactions.** (07-10, Q5) It wraps *one* method in *one* atomic transaction. It does nothing to prevent a lost update between two different HTTP requests each running their own transaction. That needs optimistic (`@Version`) or pessimistic (`SELECT ... FOR UPDATE`) locking instead.
- **`@Transactional` does NOT roll back on checked exceptions by default.** (07-16, Q5) Only unchecked exceptions (`RuntimeException`/subclasses) and `Error` trigger automatic rollback. A checked `IOException` thrown mid-method lets the transaction **commit** — silent data corruption risk (e.g., debit without matching credit). Fix: `@Transactional(rollbackFor = Exception.class)`.
- **A `UNIQUE` constraint *does* prevent duplicate-insert races.** (07-16, Q4) Stated the opposite — that duplicates could occur "although we use UNIQUE." In reality, UNIQUE is enforced atomically by the DB regardless of what the app checked first; the second concurrent `INSERT` gets rejected. Don't reach for locking here — there's no row to lock yet at insert time. This directly contradicts the *correct* answer given in 07-08 Q2, where PRIMARY KEY vs UNIQUE was reasoned about accurately — the gap re-emerged under a race-condition framing.
- **`ERR_CONNECTION_REFUSED` is not a latency symptom.** (07-04, Q3) It means the TCP connection was actively rejected (nothing listening, server down, firewall) — a fundamentally different failure mode from a slow/timeout response.
- **Applying a function to an indexed column defeats the index** (`WHERE UPPER(email) = ...`) — the DB indexed raw values, not transformed ones, so it forces a full scan. (07-05, Q1) Fix: normalize at write time (store lowercase) or use a functional index.
- **Resolving a merge conflict is not "choose one side."** (07-29, Q2) Framed conflict resolution as discussing with a teammate to pick between two competing versions. In reality, the two changes usually both need to survive — e.g. a teammate's refactor of a method's structure *and* your unrelated addition inside it. Picking one side wholesale is exactly how you silently discard real work. The correct process: read the `<<<<<<<`/`=======`/`>>>>>>>` markers, manually combine both changes into one correct version, test it, then `git add`/`commit` (or `rebase --continue`).
- **"Run the existing tests" is not the same as test-driving a bug fix.** (08-01, Q4) Asked what to do before fixing a reported bug, answered that checking/running the existing tests was enough and "we don't need to rewrite the test." But if a bug reached production, there is almost certainly no existing test covering that exact case — otherwise CI would have caught it before it shipped. The correct first step is to **write a new test that encodes the exact bug**, confirm it **fails** against current code (proof the bug is genuinely reproduced), then fix the code and confirm it **passes** (proof the fix works) — and that test then stays in the suite as a permanent regression guard. Needed two hints and still didn't reach this independently.
- **A confidently-recalled `@Transactional` rule was applied to the wrong bug.** (08-02, Q2) Given a scenario where `placeOrder` calls `this.saveOrder(order)` in the same class, and the `@Transactional` `saveOrder` doesn't roll back on failure, answered that the cause must be a checked exception needing `rollbackFor = Exception.class` — reusing the correct-in-general rule from 07-16 without checking whether it actually fit. The real cause here is **self-invocation bypassing the Spring AOP proxy**: `@Transactional` only takes effect when the call comes from outside the bean, through the proxy; a call on `this` from inside the same class never reaches it, so no transaction is opened at all and `rollbackFor` is irrelevant. This is a distinct, common Spring gotcha from the checked/unchecked rollback rule — don't conflate "the annotation exists on the method" with "the annotation is actually in effect for this call path."
- **A deadlock is not the same thing as lock contention/timeout under load.** (08-02, Q4) Given two endpoints that pessimistically lock the same two rows in opposite orders (`orders`→`inventory` vs. `inventory`→`orders`), and told this throws a real database deadlock error, answered that it was really just many concurrent users (e.g. 500) queueing for the same lock until one of them times out and mistakes it for a deadlock. That's a description of ordinary serialized waiting, not a deadlock. A deadlock is a **circular wait**: transaction A holds what B needs next and vice versa, forming a cycle with no possible progress — the database's deadlock detector finds this almost immediately (not after a load-dependent timeout) and kills one transaction to break the cycle. Fix is **consistent lock-acquisition ordering** across every code path that locks the same rows, not a load-capacity argument or a wholesale switch to optimistic locking.
- **`git rebase` does not preserve authorship any better than `git merge` does.** (08-05, Q3) Picked rebase specifically because it would supposedly let you "know who did it" for later bug-fixing help — but both merge and rebase preserve each commit's original author metadata identically; `git blame` works the same either way. The real, distinct reasons to prefer rebase (cleaner linear history, easier-to-read PR diff) don't include authorship tracking at all.
- **Deleting rows to satisfy a foreign key constraint isn't automatically the correct fix.** (08-02, Q1) Given a GDPR "delete my account" request blocked by a FK from `orders.customer_id`, proposed deleting the referencing `orders` rows first, then the customer — technically resolves the constraint, but orders are usually financial/legal records with a retention requirement that survives account deletion. The standard real-world pattern is the reverse of what was proposed: keep the orders, **anonymize the customer's PII** instead of deleting the customer row outright (or point `orders.customer_id` at a placeholder via `ON DELETE SET NULL`). Also stated the reason for keeping the FK as "it makes the query easier" rather than referential integrity (preventing orphaned rows pointing at nothing).

---

## 3. Knowledge Base by Category

### Core Java & CS Fundamentals

**Checked vs unchecked exceptions — the hierarchy, not intent, decides it.**
```
Throwable
├── Error                     (unchecked — JVM-level, e.g. OutOfMemoryError)
└── Exception
    ├── RuntimeException      (unchecked — NullPointerException, IllegalArgumentException, ...)
    └── everything else       (checked — IOException, SQLException, ...)
```
The compiler forces a method to `catch` or `throws` any checked exception because these represent *expected, recoverable* failure modes (a file might not exist, a network call might fail) — the API is telling every caller "you must decide what to do here." `RuntimeException` subtypes are left unchecked because they usually represent *programming bugs* that could theoretically occur on almost any line (a null reference, a bad cast) — requiring `throws NullPointerException` everywhere would be unworkable noise.

**Resource leaks — try-with-resources.**
```java
// Leaks the file handle if readLine() throws, and even on the happy path —
// reader.close() is never called.
public String readFirstLine(String path) {
    try {
        BufferedReader reader = new BufferedReader(new FileReader(path));
        return reader.readLine();
    } catch (IOException e) {
        return null;
    }
}

// Fixed: try-with-resources closes the resource automatically,
// even if an exception is thrown inside the try block.
public String readFirstLine(String path) {
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        return reader.readLine();
    } catch (IOException e) {
        return null;
    }
}
```
Reflex: any `Reader`/`Stream`/`Connection`/`Socket` opened with `new` inside a method → ask "does this need try-with-resources?" (It needs to implement `AutoCloseable`, which nearly all I/O classes do.)

**Big-O in practice.**
```java
// O(n × m) — 10,000 orders × 5,000 customers = 50,000,000 comparisons, ~30s
for (Order o : orders) {
    for (Customer c : customers) {
        if (o.getCustomerId().equals(c.getId())) { ... }
    }
}

// O(n + m) — build the lookup once, then a single pass
Map<CustomerId, Customer> byId = new HashMap<>();
for (Customer c : customers) byId.put(c.getId(), c);   // O(m)
for (Order o : orders) {                                // O(n)
    Customer c = byId.get(o.getCustomerId());            // O(1) average
    ...
}
```
Decision rule: use `Set<K>` when you only need "does this exist?"; use `Map<K, V>` when you need the actual object back. A common junior mixup is reaching for a `Set` of IDs when the report actually needs the full `Customer` object — that forces a second lookup pass you didn't need.

**Date/time pitfalls.**
```java
// Wrong: LocalDateTime has no timezone/offset attached at all.
// "9am" here is whatever the server's local wall clock says — meaningless
// once compared across users in different zones.
LocalDateTime sendAt = LocalDateTime.now().withHour(9).withMinute(0);

// Instant fixes "what exact moment is this" but is still not
// "9am for this specific user" — it's one single global moment.
Instant sendAt = Instant.now();

// Correct: derive from the *user's* zone, so "9am" means their 9am,
// then convert to Instant/UTC only for storage/scheduling.
ZoneId userZone = user.getZoneId();                       // e.g. stored per-user preference
ZonedDateTime local9am = ZonedDateTime.of(LocalDate.now(userZone), LocalTime.of(9, 0), userZone);
Instant sendAt = local9am.toInstant();                    // safe to store/compare across users
```
Rule of thumb: whenever `LocalDateTime.now()` touches anything cross-timezone, ask "whose 9am is this?" If the answer depends on who's reading it, you need a `ZoneId` in the picture — a UTC-safe type (`Instant`) alone doesn't solve "the right local time for this user," only "a consistent global timestamp." (Independently re-derived in [answer.md](answer.md) as well.)

**`hashCode`/`equals` contract — why overriding one without the other silently breaks `HashSet`/`HashMap`.** (07-29, Q1 — reached only after a hint)
```java
class Customer {
    Long customerId;
    @Override
    public boolean equals(Object o) {              // overridden: compares by customerId
        if (!(o instanceof Customer c)) return false;
        return Objects.equals(customerId, c.customerId);
    }
    // hashCode() NOT overridden — still Object's default (identity-based)
}
```
`HashSet.add()` does **not** compare the incoming object against every existing element with `.equals()`. It first calls `hashCode()` to compute a **bucket index** (`hash % numBuckets`), and only compares `.equals()` against whatever is already sitting in that *same* bucket. Two `Customer` instances with the same `customerId`, loaded from two different feeds, are two different objects in memory — with the default identity-based `hashCode()`, they get two different hash codes and (almost certainly) land in different buckets, so `.equals()` — which correctly says they're the same customer — never even gets called. The contract, stated explicitly: *"if two objects are equal via `.equals()`, they must return the same `hashCode()`."* Reflex: any time you override `equals()`, generate `hashCode()` in the same action (most IDEs do both together) — never one alone.

**Mutating a `HashMap` key after insertion — bucket location is fixed at `put()` time, never recomputed.** (08-03, Q3, scored 5/10 — needed two hints)
```java
class RequestKey {
    Long userId;      // mutable
    long timestamp;   // mutable
    // equals()/hashCode() derived from both fields
}

Map<RequestKey, Response> cache = new HashMap<>();
RequestKey key = new RequestKey(42L, 1000L);
cache.put(key, response);   // bucket chosen from hashCode() computed RIGHT NOW

key.timestamp = 2000L;      // mutate the same object, still sitting in the map

cache.get(key);              // hashCode() is now DIFFERENT -> searches the WRONG bucket -> returns null
```
`HashMap` computes `hashCode()` once, at `put()` time, to pick a bucket — the entry then sits in that bucket permanently; nothing rehashes it later just because the key object changed. `get()` recomputes `hashCode()` fresh on whatever key you hand it and only searches the bucket that *current* hash points to. Once `RequestKey`'s hash-relevant fields change, `get()` lands on a different bucket than the one the entry actually lives in — the object isn't lost or corrupted, it's just unreachable. `String` never hits this because it's immutable (and Java caches the computed hash inside the `String` instance after the first call — safe to do only because it can never change afterward). Reflex: any class used as a `Map`/`Set` key needs every `equals()`/`hashCode()`-relevant field to be effectively immutable for the object's lifetime as a key — mutate one, and lookups silently break with no exception, no warning.

**Integer caching / autoboxing — `==` "works" by accident for small values, a fresh gap not reached even with two hints.** (08-06, Q1, scored 3/10)
```java
Integer a = 100;
Integer b = 100;
System.out.println(a == b);   // true

Integer c = 200;
Integer d = 200;
System.out.println(c == d);   // false
```
`Integer a = 100` autoboxes through the static factory method `Integer.valueOf(int)`, not `new Integer(100)`. `valueOf()` maintains an internal cache (`IntegerCache`) of `Integer` objects for values from **-128 to 127** — any value in that range returns the *same cached object* every time, so `==` (a reference comparison) happens to return `true`. Outside that range, `valueOf()` allocates a brand-new object on every call, so two separately-boxed `200`s are different objects holding the same value — `==` correctly says "different objects," which isn't what the code meant to ask. The cache is purely a JVM optimization (small integers are extremely common — loop counters, indices), not a correctness guarantee, which is exactly why leaning on `==` here is fragile: it silently "works" for small numbers and breaks for larger ones with zero warning. Fix: **always use `.equals()`** to compare wrapper types by value, never `==`.

Reflex: any `==` next to a wrapper type (`Integer`, `Long`, `Boolean`, `Character`) is suspect until proven otherwise — most IDE static-analysis inspections flag this exact pattern.

**Also in the topic pool, not yet directly tested — review before the interview:**
- `HashMap` internals: collision handling (linked list / red-black tree since Java 8 once a bucket gets large), resizing/rehashing on load factor.
- Binary search preconditions: the collection must already be sorted.

---

### OOP, Design Patterns & LLD

**YAGNI judgment — a genuine strength, keep using this question:** "What specifically is going to vary, and how often?" If you can't name a concrete future change, a pattern is premature. Repeatedly and correctly flagged Strategy+Factory as overkill for 2-3 static, unchanging branches (07-08 Q3, 07-10 Q3, 07-14 Q3).

**Precise pattern definitions — the recurring weak spot. Identify a pattern by the problem it solves, not its shape:**

| Pattern | Solves | NOT this |
|---|---|---|
| **Strategy** | Swap an *algorithm/behavior* at runtime via a common interface | Just "interface + multiple implementers with a shared field" — that's plain polymorphism, not specifically Strategy |
| **Facade** | Give a simple unified entry point to a *complex, multi-class subsystem* (e.g. `CheckoutFacade` coordinating `InventoryService`, `PaymentGateway`, `NotificationService`) | "Putting everything into 1 class" — that's the opposite, a **God class** anti-pattern |
| **Factory** | Centralize *object creation* so callers don't need to know which concrete class to instantiate | Just "a class with a `create()` method" without a real decision being made inside it |
| **Singleton** | Guarantee exactly one shared instance (Spring beans are singleton-scoped by default, managed by the container) | Manually writing a private constructor + static `getInstance()` in app code — in Spring, let the container do this |
| **Observer** | Decouple "something happened" from "who reacts to it" (`ApplicationEvent`/`@EventListener` in Spring) | Direct method calls between unrelated classes |
| **Adapter/Decorator** | Wrap one interface to satisfy another, or add behavior without changing the wrapped object (`InputStream` wrappers like `BufferedInputStream`) | — |

**Concrete Strategy + Factory sketch (the shape that should come out when a pattern refactor *is* justified):**
```java
public interface NotificationSender {           // named for the verb it performs
    void send(String message, String recipient);
}
public class EmailSender implements NotificationSender { ... }
public class SmsSender implements NotificationSender { ... }
public class PushSender implements NotificationSender { ... }

// Factory / or simply a Spring-injected Map<String, NotificationSender> keyed by type
public class NotificationSenderFactory {
    private final Map<String, NotificationSender> senders;
    public NotificationSender get(String type) { return senders.get(type); }
}

// Usage — no if/else at all:
notificationSenderFactory.get(type).send(message, recipient);
```
Naming rule: name a strategy interface for the **verb** it performs (`PayCalculator`, `NotificationSender`) — not a noun like `PaymentSalary`.

**Composition over inheritance — when a subclass tree breaks down.**
```java
// Breaks down once ContractorEmployee needs "hourly-style pay" AND
// "a unique tax rule" — two independent axes that don't fit one subclass cleanly.
abstract class Employee { abstract double calculatePay(); }
class SalariedEmployee extends Employee { ... }
class HourlyEmployee extends Employee { ... }

// Composition: inject the varying behavior instead of subclassing further.
interface PayCalculator { double calculate(Employee e); }
class HourlyPayCalculator implements PayCalculator { ... }
class ContractorPayCalculator implements PayCalculator { ... }  // combines hourly + special tax rule

class Employee {
    private final PayCalculator payCalculator;
    double calculatePay() { return payCalculator.calculate(this); }
}
```
The OOP pillar doing the real work here is **polymorphism** — the concrete `PayCalculator` implementation that runs is decided at runtime through the interface, without the calling code needing an if/else on employee type. `Employee` can still stay an abstract/base class for shared fields (name, id) — abstract-class-for-shared-state and interface-for-varying-behavior aren't mutually exclusive.

**LLD exercise (vending machine) — state modeling skipped entirely, and two precision gaps stacked in one design.** (08-03, Q2, scored 4/10)
```java
// What was proposed:
interface Drink { String code(); double price(); String name(); }
class Pepsi implements Drink { ... }
class Coca implements Drink { ... }
// ... Tea, Coffee the same way

class VendingManager {
    FormulaManager formulaManager;   // balance/payment handling
    DrinkFactory drinkFactory;        // matches code -> product, checks stock, dispenses
}
```
Two separate mistakes in one answer:
1. **Interface + implementers for what's only data variation.** Pepsi/Coca/Tea/Coffee don't differ in *behavior* — only in `code`/`price`/`name`. The same "what specifically is going to vary?" question already used correctly elsewhere for Strategy/Factory judgment calls applies here too: nothing behavioral varies, so `Drink` should be one concrete class (or record), and each product is an *instance* of it, not a separate implementing class.
2. **"Factory" reused for inventory/repository responsibility — a second occurrence of this exact mislabeling.** Matching a code to a slot, checking quantity, and decrementing stock on dispense isn't object creation — it's tracking and mutating stock state. That's `Inventory`/`StockManager` work. A Factory's actual job (see the pattern table above) is centralizing *which concrete object to create*, not "a class with a method that does stuff."

**What the question actually asked for and didn't get answered: explicit state.** A vending machine needs at minimum (a) the current inserted balance for the in-flight transaction (resets after purchase/refund) and (b) per-product stock counts that persist across transactions. Naming these two state buckets and which object owns each is the actual point of the exercise — "what classes exist" and "what state exists" are different questions, and the second one has to be asked explicitly rather than assumed to fall out of the class list. (Not required at junior depth, but the classic extension here is modeling the machine's own mode — `IDLE`/`HAS_FUNDS`/`DISPENSING`/`SOLD_OUT` — as the textbook example of the **State** pattern.)

Reflex: for any LLD prompt, write nouns-with-state and nouns-with-behavior as two separate lists before sketching classes — and before reaching for an interface, name the concrete thing that would actually differ between implementations.

**Tell-don't-ask / anemic domain model — correct SRP instinct, but missed the underlying principle entirely.** (08-07, Q2, scored 4/10 — no hint needed, but restructured within the wrong class)
```java
// Given: OrderService reaches into Order purely through getters and does
// the work itself — Order is a passive data bag with no real behavior.
public class Order {
    private List<OrderItem> items;
    private String status;
    // only getters/setters
}
public void shipOrder(Order order) {
    double total = 0;
    for (OrderItem item : order.getItems()) total += item.getPrice() * item.getQuantity();
    if (total > 0 && order.getStatus().equals("PAID")) order.setStatus("SHIPPED");
}
```
Correctly identified the SRP smell (one method doing two jobs) and proposed extracting a `calculatePrice(order)` helper — but that still leaves the logic living outside the class that owns the data. The principle actually being tested is **tell, don't ask**: instead of asking an object for its raw fields and deciding externally, tell the object to do the thing itself.
```java
public class Order {
    private List<OrderItem> items;
    private String status;
    public double calculateTotal() {
        return items.stream().mapToDouble(i -> i.getPrice() * i.getQuantity()).sum();
    }
    public void ship() {
        if (calculateTotal() > 0 && "PAID".equals(status)) status = "SHIPPED";
        else throw new IllegalStateException("Cannot ship order in status " + status);
    }
}
// OrderService now just calls: order.ship();
```
Why it matters beyond style: the invariant ("can only ship if paid and has a positive total") now lives in exactly one place instead of depending on every caller remembering to check it correctly — the textbook failure mode of an anemic domain model. Reflex: any time a service method computes something purely from one object's own getters with no other collaborator involved, ask "should that object just do this to itself?"

---

### Databases: SQL, Indexing & Data Modeling

**PRIMARY KEY vs UNIQUE.**
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,        -- guarantees id is unique — nothing else
  email VARCHAR(255) UNIQUE,    -- this is what actually stops duplicate emails
  name VARCHAR(255)
);
```
`PRIMARY KEY` only protects the key column. A duplicate-email bug with a `PRIMARY KEY`-only schema means the schema never had a real guarantee on `email` in the first place — an app-level "check if it exists, then insert" is not equivalent, because it isn't atomic (see below).

**Why app-level "check then insert" is not a real guarantee — the classic duplicate-insert race:**
```
Time  Request A                       Request B
t0    SELECT ... WHERE email=X        
t1                                    SELECT ... WHERE email=X
t2    (no rows found)
t3                                    (no rows found)
t4    INSERT (email=X)
t5                                    INSERT (email=X)   -- duplicate!
```
Both requests read "not found" before either writes — the check and the insert are two separate statements with a gap between them. Only a **DB-level `UNIQUE` constraint** closes this gap: it's enforced atomically at insert time regardless of what any caller believed was true a moment earlier. The second `INSERT` gets rejected with a constraint-violation exception, which the app then catches and turns into a friendly "email already taken" error. **There is no row to lock yet** at this point, so pessimistic locking (`SELECT ... FOR UPDATE`) doesn't apply here — that tool is for protecting *existing* rows during concurrent updates, not for preventing concurrent inserts of a row that doesn't exist yet.

**Index mechanics and how to defeat one.**
```sql
-- Index on email is USED — matches the raw stored value directly
SELECT * FROM users WHERE email = 'someone@example.com';

-- Index on email is DEFEATED — the DB must compute UPPER(email) for
-- every row before it can compare, so it can't use the b-tree ordering
-- built on the raw column values. Forces a full table scan.
SELECT * FROM users WHERE UPPER(email) = UPPER('someone@example.com');
```
A B-tree index keeps *raw* column values in sorted order so lookups/range scans are `O(log n)` instead of `O(n)`. Wrapping the column in any function moves the comparison to computed values the index was never built on. Fixes: normalize at write time (always store lowercase before insert/update, so the query can compare directly), use a case-insensitive collation/column type, or build a functional index (`CREATE INDEX ON users (UPPER(email))` where supported).

**Habit to build**: run `EXPLAIN` (or `EXPLAIN ANALYZE`) before guessing whether a slow query is hitting the index — look for `Seq Scan`/`Full Table Scan` vs `Index Scan`/`Index Only Scan` in the plan.

**Composite indexes and the leftmost-prefix rule — correctly reasoned from first principles.** (08-01, Q2, scored 8/10)
```sql
CREATE INDEX idx_orders_customer_status_created ON orders (customer_id, status, created_at);

-- USES the index — customer_id is the leading (leftmost) column
SELECT * FROM orders WHERE customer_id = 42 AND status = 'PENDING';

-- USES the index too — still a valid prefix (customer_id alone)
SELECT * FROM orders WHERE customer_id = 42;

-- DOES NOT use the index — status is not the leftmost column, so the
-- index's sort order (by customer_id first) can't help here. Full scan.
SELECT * FROM orders WHERE status = 'PENDING';
```
A composite index is a single B-tree physically sorted by the **first** column, then by the second column *within* each value of the first, then the third *within* that. A query has to filter on a left-to-right prefix of the indexed columns to benefit — `(a)`, `(a, b)`, or `(a, b, c)` — never `(b)` alone or `(c)` alone, because the tree was never sorted with `b` or `c` as the primary key. Missing piece: naming the rule itself ("leftmost-prefix rule") and stating the write-side trade-off explicitly — every index speeds up matching reads but adds overhead to every `INSERT`/`UPDATE`/`DELETE` on that table, so index choices should be justified by how frequent/important the query actually is, and confirmed with `EXPLAIN` before committing to add one.

**Foreign keys, referential integrity, and why "just delete the children" isn't always correct.** (08-02, Q1, scored 5/10)
```sql
CREATE TABLE customers (id BIGINT PRIMARY KEY, email VARCHAR(255) UNIQUE, ...);
CREATE TABLE orders (id BIGINT PRIMARY KEY, customer_id BIGINT REFERENCES customers(id), total DECIMAL, ...);

DELETE FROM customers WHERE id = 42;  -- fails: FK constraint violation, because orders still reference id=42
```
The FK's job is **referential integrity**: it guarantees an `orders` row can never point at a `customer_id` that doesn't exist in `customers` — not "convenience" for writing queries. Dropping the constraint to make the error go away removes that guarantee and lets orphaned rows accumulate silently.

The instinct to delete the child rows first (`DELETE FROM orders WHERE customer_id = ...` then delete the customer) is the right *mechanical* pattern for a plain parent/child cleanup — but it's the wrong call specifically when the request is a **GDPR "delete my account" request on an e-commerce order system**. Order/invoice rows are usually financial records a business is legally required to retain (accounting, tax audits) even after the customer's account is gone. The standard real-world pattern is the reverse: **don't delete the orders — anonymize the customer instead** (scrub PII on the `customers` row, or repoint `orders.customer_id` at a "deleted user" placeholder via `ON DELETE SET NULL`), so order history survives with the personal data removed. Declarative schema-level options worth knowing by name: `ON DELETE CASCADE` (auto-delete children — convenient but dangerous, a mistaken parent delete silently wipes every child row with no warning), `ON DELETE SET NULL`, `ON DELETE RESTRICT` (the default-like behavior seen here).

Reflex: whenever "delete" and "financial/regulatory data" appear in the same request, ask *"am I allowed to actually remove this row, or do I need to anonymize instead?"* before reaching for a cascade delete.

**OFFSET pagination cost — precision on the mechanism.** (08-02, Q3, scored 7/10; tested under REST & API Design, but the underlying mechanism is a B-tree/indexing fact)
```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 25 OFFSET 100000;   -- 8+ seconds
SELECT * FROM orders ORDER BY created_at DESC LIMIT 25 OFFSET 0;        -- instant
```
Even with an index on `created_at`, this typically still shows as an **Index Scan** in `EXPLAIN`, not a full/sequential scan — calling it "the index doesn't work" overstates it. The real issue: a B-tree supports jumping to a specific *key value*, not to the *Nth row by ordinal position*. To satisfy `OFFSET 100000`, the engine must walk the index entry-by-entry and discard the first 100,000 rows before it can start returning any — an `O(offset)` cost inherent to offset pagination, independent of indexing. Fix: **keyset/cursor pagination** — `WHERE created_at < <last_seen_value> ORDER BY created_at DESC LIMIT 25`, storing the last row's value as the cursor for the next page, so the query jumps straight to the right key instead of counting through discarded rows. Correctness nuance: if the cursor column isn't unique (two rows can share the same `created_at`), a single-column cursor can skip or duplicate rows across pages — use a **compound cursor** `(created_at, id)` with a matching composite index, so the tie-breaker column makes every row's position unique.

**Soft delete vs. hard delete — correct core fix and no hint needed, but the sharper trade-offs were missed.** (08-07, Q3, scored 6/10)
```sql
-- Before: unrecoverable
DELETE FROM orders WHERE id = ?;

-- After: reuses the existing lifecycle status column
UPDATE orders SET status = 'CANCELLED' WHERE id = ?;
```
Correctly chose soft delete over hard delete and named table growth as a cost. Two sharper trade-offs to add:
1. **Every query against the table now has to remember to filter out soft-deleted rows** (`WHERE status != 'CANCELLED'`) — miss that filter once, in a report or a join, and cancelled orders silently reappear as if active. This bites teams far more often than raw table size; a partial index on the active statuses neutralizes most of the performance worry from row-count growth.
2. **Overwriting a shared lifecycle status column on "delete" can destroy information needed for an accurate restore.** Resetting `status` back to a fixed value like `"PENDING"` loses whatever state the order was actually in beforehand (it might have been `"PAID"`). A separate `cancelled_at` timestamp column (nullable) alongside the lifecycle status avoids this — cancelling sets the timestamp without touching the real status, and restoring is just clearing it back to `NULL`.

A proposed auto-purge of cancelled rows after a fixed window is a legitimate retention policy, but it's a separate business/compliance decision from the performance question — framing it as "needed for query speed" overstates the row-count problem and reintroduces the original complaint (an old mistake becomes unrecoverable again) as an unstated side effect.

**Also in the topic pool, worth reviewing:**
- `LIKE '%foo'` (leading wildcard) also defeats a b-tree index, same reason as a function wrapper — the index can't binary-search on a pattern that doesn't anchor at the start.
- Normalization (1NF–3NF) vs. deliberate denormalization — and being explicit about the write-cost you accept when you denormalize.
- A full status-history table (vs. the single status column used here) when you need a durable audit trail of every past state transition, not just the current one.

---

### Spring Boot & JPA/Hibernate

**Constructor injection vs field injection — the full mechanism, not just "easier to test."**
```java
// Field injection — object exists in a half-built state (empty fields),
// then Spring reflects the dependency in afterward. Not final-able.
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
}

// Constructor injection — the field can be final: guaranteed non-null
// the instant the object exists, no partially-constructed window.
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```
What actually changes:
- **`final` + fail-fast**: a missing bean fails application *startup* immediately with a clear "no qualifying bean" error. Field injection can leave a `null` field that only blows up as an NPE later, at the first call site — much more confusing to debug.
- **Plain-JUnit testability**: `new OrderService(mockOrderRepository)` needs zero Spring context or reflection. Field injection generally forces either a full `@SpringBootTest` or reflection-based mocking to set the private field.
- **SRP visibility**: a constructor with 7 parameters is an obvious, visible code smell ("this class does too much"); field injection hides that same smell because each `@Autowired` field is invisible to the others.
- It is **not** about bean lifecycle management — Spring manages both as singletons either way; "lifecycle" is the wrong word to reach for here.

**`@Transactional` — two separate gotchas, both hit in these sessions:**
```java
// Gotcha 1: rollback default only covers unchecked exceptions + Error.
@Transactional
public void transferFunds(Account from, Account to, BigDecimal amount) {
    debit(from, amount);
    log.info("debited");        // suppose this throws a checked IOException
    credit(to, amount);          // never reached — but the transaction still COMMITS,
                                  // because IOException isn't unchecked, so no auto-rollback
}

// Fix: opt in explicitly for checked exceptions.
@Transactional(rollbackFor = Exception.class)
public void transferFunds(Account from, Account to, BigDecimal amount) { ... }
```
```java
// Gotcha 2: @Transactional wraps ONE method in ONE transaction — it does
// nothing to coordinate between TWO separate requests/transactions.
// Agent A's save and Agent B's save each already run in their own
// @Transactional-wrapped method; adding more @Transactional doesn't help.
// This is a lost-update problem — solved by locking (see Concurrency below),
// not by transaction scoping.
```
Memorize as one line: *"`@Transactional` rolls back on unchecked exceptions and `Error`s by default — checked exceptions need `rollbackFor` explicitly. And it only protects atomicity within one method's transaction, never coordination across two separate transactions."*

**N+1 problem — correctly recognized and fixed on the first try.** (08-01, Q5, scored 8/10)
```java
List<Order> orders = orderRepository.findAll();       // 1 query
for (Order o : orders) {
    o.getCustomer().getName();                          // N additional queries — one per order,
}                                                        // because customer is lazily fetched
```
Spot it in logs: one query for the parent list, then N nearly-identical queries for a related entity per row — named correctly, and correctly reasoned about DB load compounding under concurrent users. Missing piece: naming **lazy loading** explicitly as the root cause (the `@ManyToOne`/`@OneToMany` association only fires its query when accessed, once per row) — the fix (`JOIN FETCH`) was named correctly, but tying it to *why* the N extra queries happen in the first place is worth stating unprompted. Fixes: `JOIN FETCH` in a JPQL query (the one given), `@EntityGraph`, or batch fetching (`@BatchSize`) — all collapse it back down to 1-2 queries.

**Entity gotchas worth reviewing:** JPA requires a no-args constructor on `@Entity` classes; `equals`/`hashCode` on entities need care (usually base them on the business/natural key or just `id`, and handle the pre-persist `null id` case) to avoid broken behavior in `Set`s/`Map`s before the entity is saved.

**`@Transactional` self-invocation — a distinct gotcha from the rollback-exception-type rule, and a fresh gap.** (08-02, Q2, scored 3/10 — a previously-correct rule misapplied to the wrong bug)
```java
@Service
public class OrderService {
    public void placeOrder(Order order) {
        validate(order);
        this.saveOrder(order);        // plain call on `this` — NOT through the Spring proxy
        notifyWarehouse(order);
    }

    @Transactional
    public void saveOrder(Order order) {
        orderRepository.save(order);
        inventoryRepository.decrementStock(order);   // throws (e.g. stock hit zero) -> should roll back save(), but doesn't
    }
}
```
`@Transactional` doesn't work on the method itself — Spring wraps the **bean** in a dynamic proxy (JDK proxy or CGLIB) that intercepts calls to open/commit/roll back a transaction. That interception only happens when the call arrives **from outside the bean, through the proxy** (e.g. another Spring-managed bean calling `orderService.saveOrder(...)`). `this.saveOrder(order)` is a plain Java call on the raw object — it never touches the proxy, so no transaction is opened for that call at all, and nothing about `rollbackFor`/exception type matters, because there's no transaction to roll back in the first place. This is easy to misdiagnose as the checked-vs-unchecked rollback issue (see Critical Misconceptions above — `@Transactional` does NOT roll back on checked exceptions by default) because both bugs present as "`@Transactional` didn't roll back," but the fix is completely different: extract the method into a **separate injected bean** so the call comes from outside the class and passes through the proxy.
```java
@Service
public class OrderService {
    private final OrderPersistenceService persistenceService;
    public void placeOrder(Order order) {
        validate(order);
        persistenceService.saveOrder(order);   // external call through the proxy -> transaction actually applies
        notifyWarehouse(order);
    }
}
@Service
public class OrderPersistenceService {
    @Transactional
    public void saveOrder(Order order) { ... }
}
```
Reflex: any time `@Transactional` "isn't working," ask *"is this method self-invoked — called via `this.method()` or a bare unqualified call from another method in the same class?"* before touching `rollbackFor` or exception types at all.

**`LazyInitializationException` — a third distinct `@Transactional`-adjacent gotcha, reached only after two hints.** (08-05, Q1, scored 6/10)
```java
@Service
public class OrderService {
    @Transactional
    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();   // Customer is lazy — not loaded yet
    }   // <-- transaction commits and Session closes HERE, the instant this method returns
}

@RestController
public class OrderController {
    public OrderResponse getOrder(@PathVariable Long id) {
        Order order = orderService.getOrder(id);
        return new OrderResponse(order.getCustomer().getName());  // throws LazyInitializationException:
    }                                                             // no Session left to fetch Customer through
}
```
The service method itself returns successfully — the failure happens strictly later, in the controller, because the `Session` a lazy proxy needs to fetch its data through only lives as long as the `@Transactional` method's transaction. The moment that method returns, the transaction commits and the Session closes; the `Order` object handed back is now **detached**. Any association not already initialized *inside* that transaction (like the lazy `customer`) can never be initialized afterward — accessing a real field on it (`getName()`) throws. Note: accessing just the FK scalar (`order.getCustomer().getId()`) is safe even on a detached proxy, since JPA can supply the id without touching the database at all — only a non-id field triggers the failing fetch.

Two standard fixes, plus the one usually preferred in production:
```java
// Fix 1 — eagerly load the association inside the still-open transaction
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.id = :id")
Order findByIdWithCustomer(@Param("id") Long id);

// Fix 2 — map to a DTO while the transaction/session is still open,
// so the controller only ever serializes plain fields, never a lazy proxy
@Transactional
public OrderDto getOrder(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    return new OrderDto(order.getId(), order.getCustomer().getName());  // access happens IN the transaction
}
```
Anti-pattern worth knowing by name: **"Open Session in View"** — extending the Hibernate session across the whole HTTP request "fixes" this exception by masking it, at the cost of hidden N+1 queries at render time; Spring recommends disabling it (`spring.jpa.open-in-view=false`) and fixing the real cause (fetch explicitly, or use a DTO) instead. Reflex: the moment an entity leaves its `@Transactional` method, treat every non-eagerly-loaded association as off-limits.

---

### REST & API Design

**Idempotency — the core lens for "what if this gets retried?"**

| Method | Idempotent? | Why |
|---|---|---|
| `GET` | Yes | Read-only, no state change |
| `PUT` | Yes (by contract) | Setting the same full state twice yields the same end state |
| `DELETE` | Yes (by contract) | Deleting an already-deleted resource is still "gone" |
| `POST` | No (by default) | Usually *creates* something — firing it twice can create two somethings |

Test: **"If I fire this exact request twice, is the end state different?"** Same → safe to retry as-is. Different → needs an explicit guard.

```java
// Cheap first guard for a PUT-like transition that has a side effect
// (e.g. cancel triggers a refund) — check current state before acting:
if (order.getStatus() == OrderStatus.CANCELLED) {
    return;  // already cancelled, no-op — makes a retried request safe
}
order.setStatus(OrderStatus.CANCELLED);
paymentGateway.refund(order);
```
Idempotency **keys** are the heavier tool, needed when the state alone can't tell you "have I already processed this exact request" (e.g. two *different* legitimate cancel-then-recreate flows could look state-identical) — client generates a unique key per logical operation, server stores "already processed this key" and returns the original result on a retry, without re-running side effects. This is the pattern Stripe and other payment APIs use.

**Action endpoint vs. generic PUT + status field:**
```
POST /api/orders/{id}/cancel     -- explicit: signals a real action with side effects
PUT  /api/orders/{id}            -- generic: "set the whole resource to this state",
                                     hides that cancelling also refunds/releases inventory
```
An explicit action endpoint communicates intent and is the natural home for the idempotency guard above; a generic `PUT` can *technically* be made idempotent too (state-check before re-running the side effect), so the real decision driver is clarity of intent, not idempotency alone.

**Pagination — why unbounded list endpoints break:**
```
GET /api/v1/books                 -- fine at 500 rows, a slow-motion crash at 2,000,000
GET /api/v1/books?page=0&size=25  -- bounded response size regardless of table growth
```
The fix belongs at both layers: the SQL query itself needs `LIMIT`/`OFFSET` (or cursor/keyset pagination for large offsets, which avoids `OFFSET`'s "still scans and discards N rows" cost), and the API contract needs `page`/`size` (or cursor) query params so a client can never accidentally request everything at once. Full mechanism (why the offset cost grows, and the keyset-pagination fix with its tie-breaker caveat) is written up under Databases → "OFFSET pagination cost — precision on the mechanism" above.

**DTOs vs. entities** — flagged as a gap, not yet corrected in an answer: returning a `@Entity` class directly from a `@RestController` couples your DB schema to your public API contract (a schema change becomes a breaking API change), and can leak fields (password hashes, internal flags) or trigger `LazyInitializationException` when Jackson tries to serialize an uninitialized lazy association outside a transaction. A separate DTO/response record decouples the two.

**Status-code precision under a 3-outcome scenario — best single answer of the 08-03 session.** (08-03, Q1, scored 7/10)
```java
// POST /api/orders/{id}/cancel always returned 200; correct per-outcome mapping:
// success                  -> 200 OK
// order id not found       -> 404 Not Found
// order already shipped    -> 409 Conflict   <- correctly NOT 400
```
Correctly resisted the common junior default of reaching for `400 Bad Request` for the "already shipped" case — the request itself is well-formed, it's the *current state* of the resource that conflicts with the action, which is exactly what `409` means, chosen with no hint needed. What was missing: the three response bodies sketched were shaped differently case-by-case (a `response` field holding a message in one, `null` in the others) instead of one consistent envelope. A real API needs a single error/response shape produced by *every* endpoint — normally via a global `@ControllerAdvice`/`@ExceptionHandler` mapping specific exceptions (`OrderNotFoundException`, `OrderAlreadyShippedException`) to status + a consistent body, rather than each controller method hand-rolling its own if/else status logic. Also missing: a stable, machine-readable error **code** (`"code": "ORDER_ALREADY_SHIPPED"`) alongside the human message, so the frontend can safely switch on the code even if the message text changes later — it can't safely pattern-match on message strings.

**Resource modeling — path vs. query parameters, and correctly reversing the design when scope changes. Strongest answer of the 08-04 session.** (08-04, Q3, scored 8/10)
```
GET /customers/42/orders             -- one customer's orders: customer id belongs in the PATH,
                                          because it identifies which resource collection you're scoped to
GET /customers/42/orders?range=7D    -- add a query param for a filter WITHIN that scope

GET /orders?range=7D                 -- "last 7 days, ALL customers": no single customer to scope to,
                                          so the route reverts to flat + a query-param filter
```
Correctly chose the nested route for a single customer's orders, correctly used a query parameter for the date-range filter rather than another path segment, and — the strongest part of the answer — correctly reversed the route design the instant the scope changed from "one customer" to "all customers," with no hint needed. The general rule, worth stating explicitly rather than only demonstrating through the example: **path parameters identify which specific resource/sub-collection you're scoped to; query parameters filter or modify the result set within that scope.** Minor precision gaps: used "URL body" instead of the correct term **path parameter**; and `range=7D` isn't a self-documenting/standard query shape — a real API would more typically use `?since=2026-07-28` or `?days=7` so the parameter's meaning doesn't depend on parsing a custom duration syntax.

**Also in the topic pool, worth reviewing:** API versioning a field change without breaking existing clients; Jackson basics (`@JsonIgnoreProperties(ignoreUnknown = true)`, date format configuration).

---

### Concurrency & Locking

**Strongest topic overall** — two race-condition-on-a-shared-counter questions both scored 8/10, with a correct read→increment→write interleaving trace:
```
Shared field: private int count = 0;   // count++ is NOT atomic — it's 3 steps:
                                        // 1. read count  2. add 1  3. write count back

Thread A: read count (0)
Thread B: read count (0)          <- both threads see the same starting value
Thread A: write count = 1
Thread B: write count = 1          <- B's write clobbers A's — one increment is lost
```
Correct fixes, in order of preference for a simple counter:
```java
// Best for a single variable — lock-free compare-and-swap, cheaper under load
private final AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();

// Works, but blocks other threads for the duration — needed once the
// invariant spans MORE than one field (atomics can't cover multi-field consistency)
private int count = 0;
public synchronized void increment() { count++; }
```
Gap to close even in the 8/10 answers: **name the concept explicitly** ("this is a race condition — specifically a lost update") in the PR comment itself, not just describe the mechanism.

**Thread-unsafe collections under concurrent access — a distinct shape from the counter race, and a fresh, partially-reasoned gap.** (08-04, Q2, scored 5/10)
```java
@Service
public class ProductService {
    private final Map<Long, Product> cache = new HashMap<>();   // plain HashMap, not thread-safe

    public Product getProduct(Long id) {
        if (cache.containsKey(id)) { return cache.get(id); }     // check-then-act, non-atomic
        Product product = productRepository.findById(id).orElseThrow();
        cache.put(id, product);
        return product;
    }
}
```
Correctly identified that concurrent request threads hitting this shared `@Service`-singleton's `HashMap` without synchronization was the root cause — but explained the "wrong/missing entries with no exception" symptom as a DB-vs-cache staleness inconsistency, which is the wrong mechanism: there's no TTL or external update event here, so nothing is "going stale." The real mechanism is that plain `HashMap` makes **no thread-safety guarantee whatsoever** — `put()` can trigger an internal resize/rehash of its bucket array, and if two threads call `put()` at the same instant, they race on that same internal array. This can silently drop or overwrite an entry, or make a concurrent `get()` return `null` for a key that should be there — with zero warning, which is also what throws `ConcurrentModificationException` in the cases that do get caught (a fail-fast check on an internal modification counter, not a guaranteed one). Also reached for manually locking the whole `Map` as the fix, and separately proposed "cache invalidation" — neither is the standard answer here (invalidation solves staleness, not thread-safety). The idiomatic fix is `ConcurrentHashMap`, purpose-built for concurrent access without hand-rolled locking:
```java
private final Map<Long, Product> cache = new ConcurrentHashMap<>();

public Product getProduct(Long id) {
    return cache.computeIfAbsent(id, key -> productRepository.findById(key).orElseThrow());
}
```
`computeIfAbsent` also fixes a second, subtler bug in the original: `containsKey` → `get` → `put` is itself a non-atomic check-then-act (two threads can both miss the cache and both hit the DB) — `computeIfAbsent` makes the lookup-or-compute atomic per key. Reflex: any mutable `Map`/`List`/`Set` field in a Spring singleton bean touched by multiple request threads → reach for the matching `java.util.concurrent` type before writing a manual `synchronized` block.

**The most valuable drill: telling apart three distinct concurrency problem shapes.** All three were tested and each time the *wrong* tool was reached for — this is the single highest-leverage thing to fix before the real interview.

| # | Shape | Symptom | Correct fix | Wrong fix reached for |
|---|---|---|---|---|
| 1 | **In-process race** — one shared mutable field, multiple threads in the same JVM | Lost updates on a counter under concurrent load | `AtomicInteger` / `synchronized` | — (this one was answered correctly) |
| 2 | **Lost update across two transactions** — two separate HTTP requests each read-then-write the same DB row | Agent A's edit silently overwritten by Agent B's save 5 seconds later | Optimistic locking (`@Version`) or pessimistic locking (`SELECT ... FOR UPDATE`) | `@Transactional` — wraps one method, doesn't coordinate two separate transactions |
| 3 | **Duplicate-insert race** — two requests both pass a "does this exist?" check before either inserts | Two rows with the same email after concurrent signups | DB-level `UNIQUE` constraint (no lock needed — nothing to lock yet) | Pessimistic locking — there's no existing row to `SELECT ... FOR UPDATE` |
| 4 | **Deadlock via lock-order inversion** — two transactions each pessimistically lock the same two+ rows, but in opposite orders | Both transactions error out with a real DB deadlock exception, near-instantly, under only moderate concurrent load | Enforce one consistent lock-acquisition order across every code path that locks those rows | Explaining it as ordinary lock contention/timeout under load, or switching wholesale to optimistic locking |

Ask first, every time: **"Is this racing inside one process (needs `synchronized`/atomics), or racing across time via the database (needs a schema constraint or transactional locking)?"**

**Optimistic locking, concretely:**
```java
@Entity
class Customer {
    @Id Long id;
    String phone;
    String ticketStatus;
    @Version
    Long version;   // Hibernate adds "AND version = ?" to every UPDATE,
}                   // and bumps it by 1 on every successful write

// UPDATE customer SET phone=?, ticket_status=?, version=? WHERE id=? AND version=?
// If Agent B already bumped the version, 0 rows match -> Hibernate throws
// OptimisticLockException. Catch it: retry the operation or report the conflict to the user.
```
Fits read-heavy, low-conflict situations (rare simultaneous edits) — no blocking, cheap, but requires handling the conflict exception.

**Pessimistic locking, concretely:**
```sql
BEGIN;
SELECT * FROM seats WHERE id = ? FOR UPDATE;   -- takes a row-level lock;
                                                 -- any other transaction locking
                                                 -- this same row must WAIT
-- ... book the seat ...
COMMIT;                                          -- lock released
```
Fits short, high-conflict operations (e.g. last-seat booking) where losing the conflict is expensive — trades throughput for safety: concurrent requests queue up and wait, which can cause timeouts under heavy load, and introduces deadlock risk if two transactions lock multiple rows in different orders. Keep the transaction short.

**Deadlock via lock-order inversion — confused with ordinary lock contention/timeout, a genuine misconception.** (08-02, Q4, scored 3/10)
```java
// Endpoint A: refund flow
Order order = orderRepository.findByIdForUpdate(orderId);       // locks orders first
Inventory inv = inventoryRepository.findByIdForUpdate(inventoryId); // then inventory

// Endpoint B: shipment-receipt flow
Inventory inv = inventoryRepository.findByIdForUpdate(inventoryId); // locks inventory first
Order order = orderRepository.findByIdForUpdate(orderId);           // then orders
```
```
Txn A: locks orders(7)                                          ─┐
Txn B: locks inventory(3)                                        ─┤  (happens ~simultaneously)
Txn A: tries to lock inventory(3) -> BLOCKS, waiting on Txn B
Txn B: tries to lock orders(7)    -> BLOCKS, waiting on Txn A     -- circular wait: neither can proceed
```
This is a **deadlock**, not lock contention: each transaction holds exactly what the other one needs next, forming a cycle with no possible progress. It is easy to mistake for "many concurrent users queueing for the same lock until one times out" — but ordinary contention is just serialized waiting in *one direction* and eventually resolves; it never gets flagged as a deadlock by the database and isn't timing/load-dependent in the same way. A real deadlock is found by the database's deadlock detector almost immediately (milliseconds, not a load-dependent timeout), which then kills one of the two transactions outright to break the cycle. The fix is **not** switching locking strategy wholesale (e.g. to optimistic locking) — it's enforcing **one consistent lock-acquisition order** across every code path that touches the same rows (e.g. always lock `orders` before `inventory`, or always lock by ascending row ID). With a single consistent order, every transaction queues behind the other in the same direction, so a cycle can never form. Reflex: any time two-or-more `SELECT ... FOR UPDATE` calls happen in one transaction, check every *other* method that locks those same tables for a different acquisition order — that's a deadlock waiting to happen, independent of load level.

**Distributed locking — partially reasoned, mechanism not yet explicit.** (08-01, Q1, scored 5/10) Given a scenario where a teammate added `synchronized` to fix duplicate order processing, and duplicates kept happening once there were two app instances: correctly identified "multiple servers" as the root symptom, and volunteered idempotency as a fix direction unprompted — but didn't explain *why* `synchronized` specifically fails, and needed a hint to recall what `synchronized` does at all.

The mechanism worth stating explicitly: a `synchronized` lock is tied to an object living inside **one JVM process**. Two app instances are two separate JVMs, each with its own independent copy of that lock object — neither has any visibility into the other's lock. A thread on instance A and a thread on instance B can each acquire "their own" copy of the lock and enter the method at the exact same instant. In-process tools (`synchronized`, `AtomicInteger`) only ever coordinate threads *within a single process* — once there's more than one app instance, they stop working entirely, silently, with no error.

The Redis `SET key value NX PX <ttl>` pattern (set-if-not-exists with an expiry) is the basic idea for a cross-instance lock — Redis (or another shared external store) becomes the one coordinator every instance defers to, instead of each instance holding its own local lock. And often you don't need a lock at all: a unique constraint, or an atomic `UPDATE ... WHERE version = ?`, already gives you the guarantee without any explicit locking machinery — usually the cleaner fix for "the same webhook/request might get delivered twice."

---

### Testing & Debugging

**Consistently the weakest category** (3/10, 4/10, 4/10, 3/10 across four sessions) — the same underlying scenario shape recurred each time: an intermittent, timing-dependent bug ("double-click submit twice fast," "occasionally in prod") that can't be reproduced by normal manual testing.

**The correct approach, step by step — the sequence itself is the thing being tested:**
1. **Reproduce reliably first.** A debugger, and honestly most tools, are close to useless until the bug can be triggered on demand. If the report includes a trigger ("double-click," "fast," "sometimes"), treat that as the literal repro recipe and recreate it mechanically — e.g. script two near-simultaneous requests (Postman runner, a small script, or a debugger breakpoint that pauses request A mid-flight while request B runs) rather than clicking a button quickly by hand, which rarely hits the exact race window.
2. **Be deliberate about tool choice for timing bugs.** A debugger can be the *wrong* first tool here — pausing execution at a breakpoint changes the timing of everything around it, which can make a race condition disappear entirely (a "Heisenbug"). Logging is safer as a first move because it observes without altering timing, and can run in production where the bug actually happens.
3. **Use structured logging, not `System.out.println` + redeploy.** `println` has no log levels (can't turn it up/down), isn't searchable/aggregable across instances, and requires a full redeploy for every adjustment. Prefer a real logger with levels (`DEBUG`/`TRACE` for detail that's normally off) and structured context — request ID, timestamp, user/order ID, thread/instance ID — so a specific failing request can be isolated out of a huge log dump. (The instinct to filter logs by a request/correlation ID was already present and correct.)
4. **Form an explicit hypothesis for *why* it's intermittent** before reaching for a tool — don't jump straight to "let's add logging" with no theory. Candidate hypotheses worth naming out loud for a "sometimes" bug: a non-idempotent endpoint being hit twice (double-submit/double-charge), an exception being silently swallowed inside a loop (job "skips" some items with no trace), or a job running on two instances/threads simultaneously.
5. **Binary-search debugging** once you have a hypothesis and can reproduce: narrow down *where* in the flow the bug is introduced by checking state at successively narrower points, rather than reading the whole flow top to bottom hoping to spot it.

**Not every intermittent bug is a timing/concurrency race — resource/threshold effects need a different hypothesis.** (08-02, Q5, scored 3/10 — a fresh gap; the 5-step framework above is specifically for *timing*-based intermittency and doesn't transfer automatically)
```java
public List<Category> flatten(Category node) {
    List<Category> result = new ArrayList<>();
    result.add(node);
    for (Category child : node.getChildren()) {
        result.addAll(flatten(child));   // recursive call per child
    }
    return result;
}
```
Scenario: one customer's account, with an unusually deep legacy-imported category tree, throws `StackOverflowError` intermittently — maybe 1 in 5 times — with **no code changes and an identical tree** between runs. The answer given only restated the obvious surface cause ("the tree is too deep") without addressing the actual question: if the input is identical every time, why does it fail only *sometimes*? That's a **resource/threshold bug**, not a logic bug — the recursion depth for this account's tree sits right at the edge of the JVM's available stack space, and small, incidental variations in how much stack the surrounding call chain has already consumed before reaching this method (which thread from a pool handled the request, how many filter/proxy layers ran first) tip it over the edge on some runs and not others.

The debugging approach this calls for is different from the timing-bug framework above: form the hypothesis "is this a resource/threshold effect (stack depth, memory, connection-pool exhaustion) rather than a timing race?" — then confirm it directly (log the recursion depth reached, or the tree's actual depth, and compare against the JVM's known default stack size) before touching a fix. `println`/logging is still the right tool to gather that evidence, but the *goal* of the logging is different: proving a threshold was crossed, not narrowing down a race window. The real fix is structural, not more logging: convert the recursion to an **iterative** traversal using an explicit stack (`Deque<Category>`) instead of the language call stack — this removes the dependency on JVM stack size entirely and scales to arbitrary depth. Raising `-Xss` (the stack size) only raises the threshold; it doesn't remove the fragility.

**Mocking anti-pattern: over-mocking / "testing the mock, not the code."** (07-29, Q5 — needed a hint and a full explanation before the concept could be restated; weakest score of that session)
```java
@Test
void appliesDiscount() {
    ProductRepository mockRepo = mock(ProductRepository.class);
    PricingEngine mockPricing = mock(PricingEngine.class);
    DiscountCalculator mockCalculator = mock(DiscountCalculator.class);
    when(mockRepo.findById(1L)).thenReturn(product);
    when(mockPricing.getBasePrice(product)).thenReturn(100.0);
    when(mockCalculator.calculate(100.0, 0.1)).thenReturn(90.0);   // stubbed to return 90.0

    DiscountService service = new DiscountService(mockRepo, mockPricing, mockCalculator);
    double result = service.applyDiscount(1L, 0.1);
    assertEquals(90.0, result);   // ...and the assertion checks for exactly 90.0
}
```
Every collaborator `DiscountService` depends on is mocked, and every mock is stubbed to return exactly the value the final assertion checks for. The test isn't verifying that `DiscountService` computed anything — it's verifying that Mockito faithfully returned the `90.0` it was told to return three lines earlier. The tell: if you deleted `DiscountService`'s real implementation and replaced it with `return 90.0;` directly, this exact test would still pass — a test that survives deleting the logic it's meant to verify isn't testing that logic. Two ways to fix it: (a) add `verify(mockPricing).getBasePrice(product)` / `verify(mockCalculator).calculate(100.0, 0.1)` to confirm the *orchestration* — that the service calls its collaborators with the right arguments in the right order — or (b) recognize that mocking every single dependency is itself the smell, and test the real discount math against a real (or minimally faked) `DiscountCalculator` instead of stubbing the answer directly. Reflex: if every dependency is mocked *and* the stub's return value matches the assertion, ask "what real behavior is left to fail if I break something?" — if none, the test isn't testing that class.

**Test-driving a bug fix — the weakest score of this batch, and a fresh gap.** (08-01, Q4, scored 3/10 — needed two hints and still didn't land on it)

Given a reported bug (cancelling an already-shipped order still triggers a refund) and asked what to do *before* touching the fix: answered "check/run the existing tests," reasoning that "we don't need to rewrite the test." This misses the actual point — if a bug reached production, there is almost certainly **no existing test** covering that exact case, because if there were, it would have failed in CI before shipping.

The correct sequence:
```java
// 1. Write a NEW test that encodes the exact reported bug
@Test
void cancelOrder_whenAlreadyShipped_doesNotTriggerRefund() {
    Order order = anOrderWithStatus(OrderStatus.SHIPPED);
    orderService.cancel(order.getId());
    verify(paymentGateway, never()).refund(any());
}
// 2. Run it against the CURRENT (buggy) code — it should FAIL.
//    That failure is proof the real bug is reproduced, not a lookalike.
// 3. Make the actual fix.
// 4. Re-run the same test — it should now PASS. Proof the fix works.
// 5. Keep the test in the suite — it's now a permanent regression guard.
```
Why the order matters, stated explicitly: a failing test *before* the fix proves the bug is genuinely reproduced in a repeatable, automated way (not just "I read the code and it looks wrong"); a passing test *after* proves the fix actually addresses it; and the test then catches this exact bug forever if a future refactor reintroduces it — instead of another support ticket months later. This is the single highest-priority testing gap to close before the real interview (see Priority List).

**Flaky test diagnosis, weak assertions, and one-behavior-per-test — a genuine non-attempt.** (08-04, Q1, scored 3/10 — extensive clarifying questions asked, but no independent answer even after a hint)
```java
@Test
void testDiscount() {
    Order order = new Order();
    order.addItem(new Item("Widget", 100.0));
    order.addItem(new Item("Gadget", 50.0));
    order.applyCoupon("SAVE10");
    assertTrue(order.getTotal() < 200.0);        // passes even with ZERO discount applied
    assertNotNull(order.getAppliedCoupon());
    assertEquals(2, order.getItems().size());    // fails ~1/20 CI runs, no local repro, no code changes
}
```
Three distinct problems stacked in one PR-review scenario, none of which were reached independently:
1. **One test, three behaviors.** The method name (`testDiscount`) doesn't say which behavior is under test, and it checks three unrelated things (total, coupon stored, item count) in one method — a failure requires reading the stack trace closely just to know *what* broke. Fix: one behavior per test, named for that behavior (`appliesCoupon_reducesTotalByExactPercentage()`), item-count assertions don't belong in a discount test at all.
2. **The assertion doesn't test what it claims to.** `assertTrue(total < 200.0)` would pass even if the coupon did nothing — pre-discount total is already $150. A loose bound like this gives false confidence. Fix: assert the exact expected value with a small tolerance — `assertEquals(135.0, order.getTotal(), 0.001)`.
3. **A flaky assertion with zero code changes and no local repro is a test-isolation smell, not a coincidence.** A freshly constructed `new Order()` should never end up with a 3rd item from code this method never calls. The likely cause is shared mutable state leaking across tests — a static/shared item catalog, a fixture reused across test methods without resetting, a test DB not rolled back between runs, or parallel test execution touching shared state. The fix is finding and eliminating that shared state (fresh data per test, proper `@BeforeEach` reset) — never loosening the assertion or re-running until green, since that just hides the bug and erodes trust in the whole suite.

Reflex for "test fails sometimes, no code changes, can't repro locally": ask *"is this test's setup actually isolated, or could it be touching something another test also touches?"* before assuming it's a timing race — plain object mutation bugs in tests are usually about shared fixtures, not concurrency.

**Unit vs. integration tests — correct restructuring, imprecise general rule, self-corrected once mid-answer.** (08-05, Q2, scored 7/10)
```java
// Given: a "unit test" for pure calculation logic that boots the whole app
// and hits a real DB — 8 seconds per test, 15 minutes of CI for ~200 of them
@SpringBootTest
class PriceCalculatorTest {
    @Autowired PriceCalculator calculator;   // real bean, real DB-backed PriceRepository
    @Test void calculatesDiscountedPrice() { ... }
}

// Restructured — no app context, no real DB, isolated logic only
@ExtendWith(MockitoExtension.class)
class PriceCalculatorTest {
    @Mock PriceRepository priceRepository;
    @InjectMocks PriceCalculator calculator;
    @Test void calculatesDiscountedPrice() {
        when(priceRepository.findBasePrice(any())).thenReturn(100.0);
        assertEquals(90.0, calculator.calculate(...), 0.001);
        verify(priceRepository).findBasePrice(any());   // also verify orchestration, not just the number
    }
}
```
Correctly identified the fix (swap `@SpringBootTest` for `@ExtendWith(MockitoExtension.class)` + mocks) and what a unit test should check: internal logic/branching, that it calls its collaborators correctly (`verify()`), and exception paths — none of which need a real DB or app context. The general rule volunteered first ("use a unit test when the method has a lot of business logic") is the wrong lens — almost every method has *some* logic. Self-corrected unprompted to "if there's a DB call or API call, I'll use an integration test," which is much closer to the real distinguishing question: **what boundary is actually being verified** — pure logic in isolation (mock every collaborator) → unit test; a real interaction across a boundary (does this query actually return the right rows against a real schema) → integration test. Missing piece: the **testing pyramid** shape itself — many fast, isolated unit tests at the base, a smaller number of integration tests validating real boundaries, very few end-to-end tests at the top. The fix here isn't "never use `@SpringBootTest`" — it's "use it sparingly for the few things that genuinely need a real DB/context, and keep pure-logic classes on fast unit tests," with the repository query itself still getting its own (much smaller number of) integration test somewhere else.

Reflex for any test: ask *"which collaborators am I trusting vs. which one am I actually verifying?"* Everything trusted gets mocked; only the thing under test runs for real. If the honest answer is "all of them, including the DB," that's an integration test by design — fine, just don't write 200 of them.

**Also in the topic pool, worth reviewing:** reading a stack trace to find the root cause among framework noise.

---

### Web Security & Frontend Basics

**CORS — improved clearly across sessions (6 → 5 → 7, most complete on 07-14). Full mechanism:**

```
localhost:3000 (frontend)  ---fetch()--->  localhost:8080 (backend)
```
Different **port** already means a different **origin** (scheme + host + port all have to match). This triggers the browser's **same-origin policy**. The critical nuance, correctly landed on by the strongest answer: **the request still reaches the backend and the backend still processes it** — what the browser blocks is letting your frontend JavaScript *read the response*, unless the server's response headers explicitly allow it.

```java
// Global config — one of the two standard Spring fixes
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:3000")   // explicit whitelist, not "*"
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*");
    }
}

// Or per-controller:
@CrossOrigin(origins = "http://localhost:3000")
@RestController
public class OrderController { ... }
```
**Preflight `OPTIONS` request**: for "non-simple" requests (JSON body, custom headers, methods beyond simple GET/POST-form), the browser first sends an automatic `OPTIONS` request asking permission before sending the real one. If the backend doesn't answer that correctly, the browser never sends the real request at all — this is exactly why a blocked call can show **zero log lines** in the controller, even though it looks like "the request failed."

**Production gotcha:** `allowedOrigins("*")` combined with `allowCredentials(true)` is rejected by browsers outright (you can't wildcard origins while also allowing cookies/credentials) — use an explicit domain whitelist in production, `*` is dev-only convenience at best.

**Request lifecycle — the weakest single answer across all 35 questions (2/10, zero partial credit). Full sequence for typing a URL and hitting Enter:**
1. **DNS resolution** — domain name → IP address (browser cache → OS cache → resolver → root/TLD/authoritative name servers).
2. **TCP handshake** — SYN → SYN-ACK → ACK, establishing a reliable connection before any data is exchanged.
3. **TLS handshake** (if HTTPS) — certificate exchange and verification, encryption keys negotiated, *before* any HTTP request is sent over the connection.
4. **HTTP request/response** — browser sends the request line + headers, server responds, browser parses and renders.

`ERR_CONNECTION_REFUSED` happens at step 2 — it means the TCP handshake was actively rejected (nothing listening on that port, service down, or a firewall blocking it). This is a fundamentally different failure from a *timeout*, which would mean the packet went unanswered rather than being refused. Practical way to make this concrete: DevTools → Network tab → click a request → "Timing" breakdown shows the real phases (DNS lookup, Initial connection, TLS, Waiting/TTFB, Content download) with real numbers.

**React data-fetching — named at the UI-component level (spinner, error component) but not yet at the hook level. Full pattern to rehearse:**
```jsx
function OrderList() {
  const [orders, setOrders] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/orders')
      .then(res => { if (!res.ok) throw new Error(res.status); return res.json(); })
      .then(data => setOrders(data))
      .catch(err => setError(err))
      .finally(() => setLoading(false));
  }, []);   // empty dependency array — run once on mount

  if (loading) return <Spinner />;
  if (error) return <ErrorComponent message={error.message} />;
  return <OrderTable orders={orders} />;
}
```
Practice naming the `useState` variables first, then which components consume them — the UI-component answer alone (spinner + error component) skips the actual state machinery being tested.

**SQL injection — correctly named, with a genuinely concrete attack, but the fix needs more precision.** (07-29, Q4 — hint used)
```java
// Vulnerable: string concatenation lets user input become part of the SQL text itself
String query = "SELECT * FROM products WHERE name = '" + request.getParameter("name") + "'";

// Fixed: PreparedStatement — structure and data are sent to the DB separately
String query = "SELECT * FROM products WHERE name = ?";
PreparedStatement stmt = connection.prepareStatement(query);
stmt.setString(1, request.getParameter("name"));
ResultSet rs = stmt.executeQuery();
```
Correctly identified that a malicious `name` value could break out of the string literal and inject additional SQL — including a concrete `JOIN`-based attack to exfiltrate data from unrelated tables (e.g. `orders`/`users`), not just a generic "it's insecure." The piece to add: **why** `PreparedStatement` fixes it — the query structure is compiled by the database *before* the value is bound, so user input is never re-parsed as SQL syntax, no matter what characters (`'`, `;`, `--`) it contains. Also worth stating explicitly: server-side input validation is fine as defense-in-depth but is **not** a substitute for parameterization — blacklist-style validation is routinely bypassed; the prepared statement is what actually closes the hole.

**Cookie tampering vs. JWT signing — right disagreement and right tool named, but the specific vulnerability and the actual fix mechanism were both imprecise.** (08-06, Q3, scored 6/10)
```
// Proposed: user=John;role=admin  stored directly in a plain cookie
```
Correctly disagreed with storing role/identity directly in a plain cookie, and correctly reached for JWT unprompted. Two precision gaps:
1. **Named the wrong risk.** Framed the danger as "a hacker can see personal information" (confidentiality/eavesdropping) — but the sharper, more direct risk in this exact scenario is **tampering**: since the cookie lives in the user's own browser, that *same logged-in user* can open DevTools and edit `role=admin` onto their own cookie to self-escalate privileges. That's an integrity problem, not a confidentiality one, and it doesn't require an outside attacker at all.
2. **Named the tool, not the mechanism.** "Attach it to the header and validate server-side" isn't what actually prevents tampering — a JWT is **cryptographically signed** (HMAC/RSA). If a client edits any claim (e.g. `role`) without the server's secret key, the signature no longer matches what the server recomputes on verification, and the request is rejected outright. Plain "server-side validation" with no signature check would have the exact same tampering hole as the raw cookie.

Missing entirely: the other standard answer to "how do you keep someone logged in across requests" — a traditional **server-side session**, where the cookie holds only an opaque random session ID and the actual `role`/user data lives server-side (DB/Redis), looked up on each request. JWT and server-side sessions are the two standard answers here; naming both — plus the tradeoff (JWT avoids a per-request lookup but can't be revoked instantly; a session ID is trivially revocable but costs a lookup) — is the complete answer.

Reflex: for any client-held data question, ask *"can the client edit this, and would the server even notice?"* — if no, that's the vulnerability, whether it's a cookie, a hidden form field, or a request header.

**Also worth reviewing:** never trusting client-side-only validation (hiding a button isn't security — always re-validate server-side); traditional server-side sessions (opaque session ID + server-side lookup) as the sibling answer to JWT — named as a gap above, not yet volunteered independently.

---

### System Design & Architecture Basics

**Layered architecture — a genuine, consistent strength.** Controller (API surface) → Service (business logic) → Repository (persistence), with clear ownership boundaries described accurately across every session it came up in.

**Caching — best system-design answer of all sessions (8/10). Cache-aside, fully worked example:**
```
1st request:  client -> app -> (cache MISS) -> DB -> app writes to cache -> client
2nd request:  client -> app -> (cache HIT)  -> client   (DB never touched)
```
The strong, concrete example given was inventory overselling under a too-long TTL: cached "5 in stock" goes stale while real stock actually hits 0, and without a `quantity < 0` guard, two more sales get accepted against a phantom cached count. Missing vocabulary to add:
- **Cache stampede / thundering herd** — the too-short-TTL failure mode: many requests miss the cache at the exact same instant (e.g. right as a hot key expires) and all hit the DB simultaneously, which is exactly what caching was supposed to prevent.
- **Invalidate-on-write** as the complement to TTL — the moment stock/price actually changes, evict or update the cache entry immediately rather than waiting for the TTL to expire naturally. Rule of thumb: long TTL for slow-changing fields (description, images), short-TTL-or-invalidate-on-write for fast-changing/high-stakes fields (stock, price).

**Bottleneck reasoning — needs a repeatable heuristic.** The one question that reliably locates it: **"which operation happens far more often, reads or writes?"** Missed on the URL-shortener question — redirect/lookup traffic vastly outweighs create traffic (often 100:1+) for something like a URL shortener, so the bottleneck is the read/redirect path, and the fix is a cache (e.g. Redis) in front of the DB for hot short codes, plus a unique index on `short_code` so the lookup itself isn't a table scan. Redirect endpoint mechanics: HTTP `301` (permanent) or `302` (temporary) status with a `Location` header pointing at the long URL.

**CRUD schema design — good instinct, one consistent process gap:** walk every planned endpoint against the schema before finalizing it — this was missed twice (no `completed` boolean for a "mark complete" API; no list/search endpoint at all in a separate session). Checklist to run every time: **list, get-one, create, update, delete** — does the schema have what each of those five needs to read or write?

**Sync (direct call) vs async (message queue) — the recurring confusion is "non-blocking" vs "decoupled."** (07-29, Q3)
```
Direct call (sync OR async style)          Message queue
caller ---request---> callee               caller --publish("OrderPlaced")--> queue ---> consumer ---> email service
caller still needs callee's result          caller returns immediately, never waits on
or has to handle its failure right now      the consumer or the downstream service at all
```
`async/await` (JS) or a non-blocking `HttpClient` call (Java) is still a **direct call** to the downstream service — it just doesn't block the calling *thread*. The caller is still coupled to that service's availability: if it's down or slow, that failure has to be handled right there in the request path. A **message queue** (RabbitMQ/Kafka/SQS) is a stronger, different kind of decoupling — the caller publishes an event and returns immediately, and a separate consumer processes it whenever it can, even if the downstream service is down for the next 10 minutes. It also gives retries for free on transient failures, instead of a fire-and-forget async call silently swallowing them. Decision rule: **"Does the caller need the result before it can respond to the user?"** Payment → yes, must be a direct call the endpoint waits on. Confirmation email → no, perfect candidate for a queue.

**Stateless services — in-memory session state breaks the instant there's a second instance. Best System Design answer of the 08-06 session.** (08-06, Q2, scored 7/10)
```
1 instance:              login -> writes session to in-process Map -> every later request hits the same Map -> works fine
2 instances, round-robin: login -> server A's Map -> next request routed to server B -> B's Map has nothing -> "not logged in"
```
Correctly traced the exact mechanism unprompted after a single nudge: session state lives in one instance's local memory, invisible to the other instance, and the load balancer's round-robin routing is what makes the resulting logout look random rather than deterministic. Correctly proposed the standard fix — externalize the session store to Redis (or a shared DB) so every instance reads/writes the same source of truth. Missing: naming the general principle this is one instance of — **services should be stateless**; anything living in a singleton/in-process field (a session map, an in-memory cache, a counter) only exists on the instance that wrote it, so it silently breaks the moment there's more than one instance, regardless of what that field happens to be. Also missing: the alternative fix, **sticky sessions** (load balancer routes the same client to the same instance every time, via a cookie or source-IP hash) — simpler to bolt on than externalizing storage, but fragile (a restarted instance loses every session pinned to it) and can unbalance load; externalizing to Redis is the more robust default for exactly that reason.

Reflex: "works with 1 instance, breaks with 2, and involves something stored in memory" → ask *"where does that state actually live, and is it visible to every instance?"*

**Also in the topic pool, worth reviewing:** vertical vs horizontal scaling terminology itself; in-memory counters/rate-limiters hit the exact same multi-instance problem as sessions (same fix — externalize the state or make the service stateless).

---

### Tooling: Git, Build & Environment

**Commit hygiene — measurably improved and stuck across sessions (4/10 with a hint → 7/10 → 7/10).** Solid, repeatable final answer: split a mixed commit by logical concern using conventional-commit prefixes, each justified with a concrete scenario:
```
fix: reject empty bearer token in auth middleware
chore: rename OrderProcessor -> OrderPipeline
refactor: reformat build.gradle
```
Still consistently missing without a prompt — the two most important "why it matters beyond style" reasons to state explicitly every time:
- **`git blame`/`git bisect` clarity**: a future regression search (`git bisect`) or a `git blame` on any of those lines should land on a commit whose message actually describes what changed there — not on "fixes stuff," which also happens to contain two unrelated changes.
- **Revert safety**: if the auth fix turns out wrong and needs reverting, a bundled commit forces reverting the rename and the CSS tweak along with it — an all-or-nothing revert for something that should be surgical.
- Also worth naming: **review-noise/signal** — a security-sensitive change sitting next to cosmetic renames is much easier for a reviewer to skim past by accident.

Habit to state as a reflex: *"If I had to revert or `git blame` just this one change in three months, could I isolate it cleanly?"* If not, split the commit — use `git add -p` to split hunks within a single file if needed.

**Secrets in git.** Correctly identified rotation + stop committing `.env` as the fix direction; the piece consistently missing:
- **A committed secret stays in git history forever**, even after a later commit deletes the file — anyone with clone access can still find it in an old commit. This is *why* rotation is mandatory, not optional, once a secret has been pushed even once.
- Concrete mechanism: add `.env` to `.gitignore` immediately (so it can't be accidentally re-committed), and commit a `.env.example` with placeholder keys/values so onboarding still works without exposing anything real.
- Real secrets belong in a runtime-injected source — CI/CD secret store or a secret manager (Vault, AWS Secrets Manager, etc.) — never hardcoded in the repo, even in a "local-only" file that gets committed by accident.
- Bonus habit: secret-scanning in CI (GitHub secret scanning, `gitleaks`) catches this class of mistake before merge.

**Maven dependency conflicts — a genuine weak spot (3/10).**
```bash
mvn dependency:tree            # prints the resolved graph; flags dropped versions as "(conflict)"
./gradlew dependencies         # Gradle equivalent
./gradlew dependencyInsight --dependency jackson-databind
```
Maven's resolution rule is **nearest-wins**: the dependency declaration closest to your own project in the tree wins, *regardless of which version is actually newer* — a transitive dependency three levels deep can silently lose to an older version declared one level up. Fixes: pin the version explicitly in your own `pom.xml` (a direct declaration always wins over a transitive one), use `<dependencyManagement>` to centralize versions across a multi-module project, or `<exclude>` the unwanted transitive version from whichever dependency is pulling it in.

**Resolving a merge conflict — "combine both," not "pick one side."** (07-29, Q2 — see also the Critical Misconceptions entry above)
```
<<<<<<< HEAD
// your current branch's version of the method
=======
// incoming version from main (teammate's refactor)
>>>>>>> main
```
The instinct to reach for first was "discuss with the teammate and choose between the two versions" — but that's usually the wrong frame. If your teammate restructured the method and you separately added a new field or behavior inside it, the correct resolution has **both** — the new structure *and* your addition — not one or the other; picking a side wholesale is how real work quietly disappears. Full process: read both blocks inside the markers, manually edit them into one correct combined version, delete the markers, **test/run it** (a syntactically clean merge can still be semantically wrong), then `git add <file>` and `git commit` (merge) or `git rebase --continue` (rebase). An IDE's 3-way merge view (yours/theirs/result) makes "combine both" far more obvious than eyeballing raw markers in a plain editor. Before even messaging a teammate, `git log -p <file>` or `git blame` on the conflicting lines often shows exactly what each side changed and why.

**Environment config across dev/staging/prod — decent instinct, missing the Spring-specific and secrets-specific tools.** (08-01, Q3, scored 6/10)

Given a scenario where a Spring Boot app boots locally but crashes in staging with `Could not resolve placeholder 'DB_PASSWORD'`: correctly proposed splitting config by environment (e.g. `.env`, `.env.staging`, `.env.production`) — good instinct that config shouldn't be one-size-fits-all. Missing pieces:
- **The actual root cause** wasn't named: something local-only (a `.env` file, an IDE run config, an exported shell variable) supplies `DB_PASSWORD` on the laptop, and it simply was never set up in staging — the bug is implicit local machine state that didn't travel with the app, not "config" in the abstract.
- **Spring's own mechanism** for non-secret environment differences: profiles — `application-dev.properties` / `application-staging.properties` / `application-prod.properties`, switched via `spring.profiles.active=<profile>`.
- **For the secret itself**, the stronger production pattern isn't a checked-in `.env.staging` file at all — it's the deployment platform injecting the real value as a runtime environment variable (CI/CD secret store, Kubernetes Secret, AWS Secrets Manager, etc.), so it's never sitting in a file in the repo, even a per-environment one. `.env` files are fine for local dev convenience but must be `.gitignore`'d the moment they hold a real secret — commit a `.env.example` with placeholder keys instead.

Two-lens reflex: "config that differs by environment but isn't secret" → Spring profiles; "actual secrets" → platform-injected env vars / secret manager, never a repo file.

**Merge vs. rebase — mechanics needed a full explanation from scratch, then a fresh authorship misconception.** (08-05, Q3, scored 4/10 — weakest answer of the session)

Given a scenario (four days on a feature branch, five teammate commits landed on `main`, need to update before opening a PR) and asked to choose between `git merge main` and `git rebase main`: did not recall the mechanics of either without a full explanation first.

```
git merge main            git rebase main
     A---B---C (main)          A---B---C (main)
    /         \                         \
D---E (yours)  M  <- new merge commit    D'---E' (yours, replayed
                   with 2 parents           on top of C, NEW hashes,
                   (history stays          same content — straight-
                   non-linear)             line/linear history)
```
`merge` leaves your existing commits untouched and adds a new two-parent merge commit on top, tying the two histories together — non-linear, but nothing is rewritten. `rebase` sets your commits aside and replays them one at a time on top of `main`'s current tip — same content, but every commit gets a **brand-new hash**, producing a straight linear history.

Picked rebase — a reasonable instinct for this scenario — but the stated reason was a genuine misconception: that rebase "keeps the history of who did the change" so you'd know who to ask for help on a bug, implying merge doesn't. **Both merge and rebase preserve each commit's original author metadata identically** — `git blame`/`git log --author` works the same either way; rewriting a commit's hash during rebase does not touch its author field. The actual reasons to prefer rebase here are a cleaner, linear history and an easier-to-read PR diff — not authorship tracking.

When asked for the case where you'd deliberately avoid rebase, landed only on a vague "it will conflict" after a hint pointing at what happens once a teammate has already pulled and built on your original commits. The precise mechanism: because rebase mints new hashes for every commit, a teammate who already pulled your branch and committed on top of the *old* hashes is now diverged from your rebased branch at the hash level — force-pushing your rebase pulls the rug out from under their local branch (their commits' parents no longer exist upstream), risking lost or duplicated work when they try to reconcile it. This is not an ordinary merge conflict; it's a history-rewrite problem. **Rule to memorize: never rebase (and force-push) a branch other people have already pulled and built on** — rebase freely on a private branch only you use; once it's shared, prefer merge, or coordinate the rebase explicitly with everyone who has it.

Reflex before any rebase: *"has anyone else pulled this branch?"* If yes, don't rebase it — or if you must, make sure everyone re-syncs and knows it's coming first.

**Responding to PR review feedback you disagree with — evidence over assertion, a total non-attempt.** (08-07, Q1, scored 2/10 — needed two hints and still gave no independent answer)

Given a reviewer comment questioning whether a bug fix actually covers a second code path (batch import), and asked how to respond while believing the fix is already correct and unaffected: no attempt was made even after two hints, worse than the usual "needed a hint but eventually reasoned it out" pattern seen elsewhere in this log.

The correct approach: **verify before you argue.**
1. Check the code, not your memory — trace whether the other path (batch import) calls the same method/logic you changed, or a separate implementation.
2. Turn the check into evidence, not just a claim — point to the specific file/line so the reviewer can verify it themselves in seconds.
3. Better still, write a quick test exercising the questioned path with the same input that triggered the original bug — concrete proof, and it becomes a permanent regression guard either way.
4. If you turn out to be wrong, this same process surfaces that *before* you argue in the thread, not after.

Reflex: a code-review disagreement is resolved with evidence (code, tests, logs) — never by restating your own confidence level.

**Also in the topic pool, worth reviewing:** what a reviewer is actually looking for when *giving* a review (not just responding to one), and how to give constructive feedback yourself.

---

## 4. Priority List Before the Real Interview

Ranked by (a) how wrong the current understanding is, and (b) how likely it is to come up:

1. **`@Transactional` semantics** — rollback only on unchecked exceptions/`Error` by default; does not coordinate across separate transactions/requests. (Confidently wrong twice — highest priority.)
2. **`@Transactional` self-invocation bypassing the Spring proxy** — a *distinct* gotcha from #1: calling a `@Transactional` method via `this.method()` (or a bare call) from inside the same class skips the AOP proxy entirely, so no transaction is ever opened, regardless of `rollbackFor` or exception type. (08-02 — confidently misapplied the checked/unchecked rollback rule from #1 to this different bug; both present identically as "`@Transactional` didn't roll back," so telling them apart matters.)
3. **Test-driving a bug fix** — before fixing any reported bug, write a new test that encodes it and confirm it fails first; don't rely on "check the existing tests." (08-01 — weakest score of that session, needed two hints and still didn't reach it independently; ties directly into Testing & Debugging being the weakest category overall.)
4. **Resolving a code-review disagreement with evidence, not assertion** — before pushing back on a reviewer's comment, verify by tracing the actual code path in question or writing a quick test, then show that evidence rather than restating confidence. (08-07 — a genuine non-attempt even after two hints, the same "freeze instead of guessing" failure mode seen on other total non-attempts in this log; a common daily-work skill.)
5. **Merge conflict resolution** — resolving means combining both changes into one correct version, not discussing with a teammate to pick one side wholesale. (07-29 — a fresh misconception on a very common daily task.)
6. **Mocking anti-patterns (over-mocking)** — recognize when every collaborator is mocked and the assertion just checks the stub's own return value; know to verify orchestration (`verify()`) or stop mocking everything. (07-29 — weakest score of that session, needed the most support.)
7. **Telling apart the concurrency problem shapes** and their distinct fixes: in-process race → atomics/locks; **cross-instance duplicate processing → distributed lock or idempotency, not `synchronized`** (which is JVM-local only — added 08-01, partially reasoned but the mechanism wasn't yet explicit); cross-transaction lost update → optimistic/pessimistic locking; duplicate-insert race → `UNIQUE` constraint (no locking needed); **deadlock (lock-order inversion) → enforce one consistent lock-acquisition order — not a lock-contention/timeout explanation, and not a wholesale switch to optimistic locking** (08-02 — a flat misconception, mistook a real circular-wait deadlock for ordinary contention under load).
8. **Sync vs async / message queue decoupling** — a non-blocking `async/await` call is still a direct call coupled to the downstream service's availability; a message queue is a stronger, different kind of decoupling. (07-29.)
9. **Debugging methodology for intermittent bugs** — two distinct hypotheses depending on the shape: **timing/concurrency races** (reproduce first, understand why a debugger can distort timing bugs, use structured logging with correlation IDs — weakest category by average score across sessions) vs. **resource/threshold effects** (e.g. recursion depth sitting right at the JVM's stack-size limit — identical input fails only sometimes because of incidental variation in already-consumed stack, not a race). (08-02 — applied no framework at all when the bug turned out not to be timing-based; this distinction itself is the fresh gap.)
10. **Request lifecycle fundamentals** (DNS → TCP → TLS → HTTP) — scored 2/10 with zero partial credit, a very standard interview question.
11. **Integer caching / autoboxing (`==` vs `.equals()` on wrapper types)** — `Integer.valueOf(int)` caches instances for -128..127, so `==` "accidentally works" for small values and silently breaks for larger ones; always use `.equals()` for wrapper-type comparisons. (08-06 — flat non-answer even after two hints; a very common interview gotcha, high likelihood of coming up.)
12. **Precise design-pattern definitions** (Facade vs. God class; Strategy vs. plain polymorphism; Factory vs. inventory/repository responsibility — misapplied a second time on the vending-machine exercise, 08-03) — the *judgment* of over-engineering is already solid; the *vocabulary* is not.
13. **Tell-don't-ask / anemic domain model** — when a service method computes something purely from one object's own getters, the logic (and the invariant it enforces) belongs on that object, not scattered across every caller that reads its fields. (08-07 — a fresh gap; correct SRP instinct with no hint needed, but restructured within the wrong class entirely.)
14. **`hashCode`/`equals` contract mechanism** — the bucket-index lookup in `HashSet`/`HashMap`; conclusion was right after a hint, mechanism needs to be stated unprompted. (07-29.)
15. **Foreign keys vs. business/legal retention requirements** — deleting referencing rows to satisfy a FK constraint isn't automatically correct; check whether that data must be retained (financial/legal/audit — e.g. a GDPR delete on an order history table) and anonymize the parent instead of cascading the delete when it does. Know `ON DELETE CASCADE`/`SET NULL`/`RESTRICT` by name. (08-02.)
16. **Cookie tampering vs. JWT signing precision** — the risk in "role stored in a plain cookie" is the *same user* editing their own cookie to self-escalate (an integrity/tampering problem), not primarily an outside attacker reading it (confidentiality); and the reason JWT fixes it is that it's **cryptographically signed**, so a tampered claim fails signature verification — "validate server-side" alone isn't the mechanism. Also know traditional server-side sessions (opaque session ID + server-side lookup) as the sibling answer to JWT. (08-06 — right disagreement and right tool named, but both the specific vulnerability and the fix mechanism were imprecise.)
17. **Maven/Gradle dependency conflict diagnostics** (`dependency:tree`, nearest-wins) — a concrete, practical gap.
18. **Idempotency terminology** — use the word explicitly and know the PUT-safe / POST-risky default.
19. **SQL injection fix precision** — know *why* `PreparedStatement` works (structure/data sent separately) and that validation isn't a substitute for it; the vulnerability and a concrete attack were already identified correctly. (07-29 — lower priority, mostly a polish item.)
20. **Environment/secrets config across dev/staging/prod** — know Spring profiles for non-secret config and platform-injected secrets for real ones, rather than hand-distributing per-environment files. (08-01 — lower priority, decent instinct already, just needs the specific tools named.)
21. **Soft-delete trade-off precision** — the sharper costs of a status-flag soft delete are that every query must remember to filter it out (not primarily table size, which indexing handles) and that overwriting a shared lifecycle status column on "delete" can destroy the prior state needed for an accurate restore. (08-07 — correct core mechanism with no hint needed, decent priority since soft delete is a very common junior-scale pattern.)
22. **Naming and precision polish**: state the **leftmost-prefix rule** explicitly for composite indexes; name **lazy loading** explicitly as N+1's root cause; distinguish "index used, but must walk and discard offset rows" from "full table scan" when explaining OFFSET pagination cost; remember the **compound-cursor tie-breaker** (`(created_at, id)`) for keyset pagination correctness. All reasoned correctly or nearly so — lowest priority, just needs sharper naming. (08-01, 08-02.)
23. **LLD state modeling as an explicit step** — before sketching classes, separately list what state the system must track (transient vs. persistent) and who owns it; skipping this step let the entire state half of the vending-machine question go unanswered. (08-03.)
24. **Centralized/global error-response contract** — one shared envelope + stable error code produced via `@ControllerAdvice`/`@ExceptionHandler`, not a per-endpoint ad hoc shape; the status-code judgment itself (404/409 across a multi-outcome action) is already strong. (08-03 — lower priority, mostly a completeness item.)
25. **Flaky test root-causing** — a flaky assertion with no code changes and no local repro is almost always a test-isolation/shared-mutable-state problem (a fixture or catalog leaking across tests), not a coincidence to shrug off; combine with one-behavior-per-test and asserting exact expected values instead of loose bounds. (08-04 — a flat non-attempt even after a hint; Testing & Debugging remains the weakest category overall across four sessions now, so this is high-priority despite being numbered last here.)
26. **Thread-unsafe collections vs. `ConcurrentHashMap`** — a plain `HashMap`/`ArrayList`/`HashSet` field in a Spring singleton bean touched by multiple request threads is a distinct gap from the counter-race pattern already handled well; know `ConcurrentHashMap` + `computeIfAbsent` as the default fix, and that the "wrong/missing entries" symptom is internal structural corruption from concurrent `put()`/resize, not a cache-staleness problem that invalidation would fix. (08-04 — root cause partially reasoned, but conflated with staleness and reached for manual locking over the idiomatic collection type.)
27. **Merge vs. rebase mechanics, and the shared-branch force-push danger** — needed a full from-scratch explanation of what each command actually does; also a fresh confident misconception that rebase "preserves who did it" better than merge (both preserve authorship identically). The real reason to avoid rebase once a branch is shared: it mints new commit hashes, so force-pushing it after a teammate has pulled and built on the old hashes risks lost/duplicated work — not an ordinary merge conflict. (08-05 — weakest score of that session, daily-use tooling that should be automatic.)
28. **`LazyInitializationException` / entity detachment** — needed two hints to connect `@Transactional`'s transaction boundary to why the Hibernate session is already closed by the time a controller touches a lazy association returned from the service layer. Know unprompted: an entity detaches the instant its owning transaction commits; only non-id fields on an uninitialized association trigger the failing fetch. Standard fixes: `JOIN FETCH` inside the transaction, or map to a DTO before returning — plus know "Open Session in View" by name as the anti-pattern that masks rather than fixes this. (08-05.)
29. **Testing pyramid vocabulary and shape** — frame unit vs. integration around *what boundary is being verified*, not "how much business logic" a method has; know the pyramid shape (many unit tests, fewer integration tests, very few end-to-end) and that mocking a repository in one test doesn't mean the real query never gets its own, separate integration test. (08-05 — lower priority; the practical restructuring instinct was already correct, this is a vocabulary/completeness gap.)
30. **Stateless services as an explicit principle, plus sticky sessions as the sibling fix** — the practical instinct to externalize session storage to Redis was already correct and unprompted; missing is naming *why* (services should be stateless — any in-process field breaks past one instance) and knowing sticky sessions (route the same client to the same instance via cookie/IP hash) as the simpler-but-fragile alternative. (08-06 — lowest priority here, since the core answer scored well at 7/10; this is a naming/completeness gap on an otherwise strong answer.)

## 5. Consistent Strengths (keep doing these)

- Layered architecture (controller/service/repository) reasoning.
- Tracing concurrency bugs step-by-step (thread interleaving) rather than reciting definitions — this produced the two best scores across all sessions.
- Good YAGNI/over-engineering instincts on design patterns.
- Willingness to reason out loud, ask clarifying questions, and give a best guess under uncertainty rather than freezing (explicitly praised in multiple session summaries).
- Commit-hygiene instincts have measurably improved and stuck across sessions — evidence that repeated practice on a topic here is translating into consistent recall.
- Giving concrete, non-trivial attack/impact scenarios instead of generic answers — e.g. the SQL injection answer named a specific `JOIN`-based data-exfiltration exploit rather than just "it's insecure" (07-29).
- Recognizing well-known mechanisms fast and correctly on the first try when the pattern is familiar — N+1 and composite-index leftmost-prefix behavior (both 08-01) were identified and reasoned correctly from first principles, no hint needed.
- Volunteering practical, real-world fixes unprompted even when the core mechanism explanation is incomplete — e.g. bringing up idempotency as a fix direction for cross-instance duplicate processing before the `synchronized`-is-JVM-local mechanism was fully explained (08-01, Q1).
- Once the right high-level technique is identified, producing genuinely correct, concrete code for it on a brand-new scenario rather than a generic sketch — e.g. the keyset-pagination SQL (`WHERE created_at < ? ORDER BY ... LIMIT 25`) and the delete-children-before-parent ordering for a FK conflict were both essentially correct on the first try (08-02, Q3 and Q1).
- Good defensive instinct before a destructive operation: checking whether *other* tables also reference the one being deleted from, not just the obvious one (08-02, Q1).
- Precise HTTP status-code judgment under a multi-outcome scenario — chose `409 Conflict` over the common junior default of `400` for a state-conflict error, on the first try with no hint (08-03, Q1).
- Correctly reversing a design decision the instant a stated constraint changed, rather than defending the original answer — reversed a nested-vs-flat REST route the moment the scope changed from one customer to all customers, no hint needed (08-04, Q3).
- Self-correcting an initial rule mid-answer without being prompted — refined "use unit tests when there's a lot of logic" to "DB/API call in the test means integration test" on the testing-pyramid question, unprompted (08-05, Q2).
- Deriving a multi-instance statelessness bug from first principles with only a single light nudge — correctly traced why an in-memory session `Map` causes random logouts once a second instance joins a round-robin load balancer, and proposed the standard externalized-store fix (Redis) unprompted (08-06, Q2).
- Spotting a real single-responsibility smell (a method both computing a total and deciding a status transition) with zero hint needed, even though the deeper fix (tell-don't-ask) wasn't reached (08-07, Q2).
- Volunteering a concrete, unprompted follow-up concern (a data-retention/purge policy) when reasoning about a soft-delete design, rather than stopping at the schema change alone (08-07, Q3).
