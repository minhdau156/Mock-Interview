# Mock Interview — Knowledge Summary

**Covers sessions:** 2026-07-04, 07-05, 07-06, 07-08, 07-10, 07-14, 07-16, 07-29, 08-01 (9 sessions, 45 questions)
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

Trend is upward but noisy (4.0 → 6.8 with dips) — scores depend heavily on which specific angle a topic is tested from, not just topic familiarity. Best sessions (07-06, 07-14) both landed a clean 8/10 on **concurrency/race-condition tracing** and handled **CORS/system design** reasonably. Weakest sessions (07-04, 07-05) struggled most with **debugging methodology** and **request-lifecycle/exception fundamentals**. 07-29 was the first session to deliberately target sub-topics *not yet documented here* rather than seed-randomized picks — the dip to 4.6 reflects fresh, previously-untested gaps (merge conflict mechanics, mocking anti-patterns) surfacing for the first time, not regression on known material. 08-01 continued that deliberate-gap-targeting approach and landed 6.0/10 — clean, correct-on-first-try answers on composite-index reasoning and N+1 recognition, but exposed a fresh and fairly deep gap in test-driving a bug fix, plus a partial gap on why in-process locks don't coordinate across app instances.

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

**Also in the topic pool, not yet directly tested — review before the interview:**
- String immutability and why it makes `String` safe as a `HashMap` key.
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

**Also in the topic pool, worth reviewing:**
- `LIKE '%foo'` (leading wildcard) also defeats a b-tree index, same reason as a function wrapper — the index can't binary-search on a pattern that doesn't anchor at the start.
- Normalization (1NF–3NF) vs. deliberate denormalization — and being explicit about the write-cost you accept when you denormalize.
- Soft delete (`deleted_at`/`is_deleted`) vs hard delete, and status field vs. a full status-history table when you need an audit trail.

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
The fix belongs at both layers: the SQL query itself needs `LIMIT`/`OFFSET` (or cursor/keyset pagination for large offsets, which avoids `OFFSET`'s "still scans and discards N rows" cost), and the API contract needs `page`/`size` (or cursor) query params so a client can never accidentally request everything at once.

**DTOs vs. entities** — flagged as a gap, not yet corrected in an answer: returning a `@Entity` class directly from a `@RestController` couples your DB schema to your public API contract (a schema change becomes a breaking API change), and can leak fields (password hashes, internal flags) or trigger `LazyInitializationException` when Jackson tries to serialize an uninitialized lazy association outside a transaction. A separate DTO/response record decouples the two.

**Also in the topic pool, worth reviewing:** consistent error response shape (code, message, field-level errors) across all endpoints; API versioning a field change without breaking existing clients; Jackson basics (`@JsonIgnoreProperties(ignoreUnknown = true)`, date format configuration).

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

**The most valuable drill: telling apart three distinct concurrency problem shapes.** All three were tested and each time the *wrong* tool was reached for — this is the single highest-leverage thing to fix before the real interview.

| # | Shape | Symptom | Correct fix | Wrong fix reached for |
|---|---|---|---|---|
| 1 | **In-process race** — one shared mutable field, multiple threads in the same JVM | Lost updates on a counter under concurrent load | `AtomicInteger` / `synchronized` | — (this one was answered correctly) |
| 2 | **Lost update across two transactions** — two separate HTTP requests each read-then-write the same DB row | Agent A's edit silently overwritten by Agent B's save 5 seconds later | Optimistic locking (`@Version`) or pessimistic locking (`SELECT ... FOR UPDATE`) | `@Transactional` — wraps one method, doesn't coordinate two separate transactions |
| 3 | **Duplicate-insert race** — two requests both pass a "does this exist?" check before either inserts | Two rows with the same email after concurrent signups | DB-level `UNIQUE` constraint (no lock needed — nothing to lock yet) | Pessimistic locking — there's no existing row to `SELECT ... FOR UPDATE` |

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

**Distributed locking — partially reasoned, mechanism not yet explicit.** (08-01, Q1, scored 5/10) Given a scenario where a teammate added `synchronized` to fix duplicate order processing, and duplicates kept happening once there were two app instances: correctly identified "multiple servers" as the root symptom, and volunteered idempotency as a fix direction unprompted — but didn't explain *why* `synchronized` specifically fails, and needed a hint to recall what `synchronized` does at all.

The mechanism worth stating explicitly: a `synchronized` lock is tied to an object living inside **one JVM process**. Two app instances are two separate JVMs, each with its own independent copy of that lock object — neither has any visibility into the other's lock. A thread on instance A and a thread on instance B can each acquire "their own" copy of the lock and enter the method at the exact same instant. In-process tools (`synchronized`, `AtomicInteger`) only ever coordinate threads *within a single process* — once there's more than one app instance, they stop working entirely, silently, with no error.

The Redis `SET key value NX PX <ttl>` pattern (set-if-not-exists with an expiry) is the basic idea for a cross-instance lock — Redis (or another shared external store) becomes the one coordinator every instance defers to, instead of each instance holding its own local lock. And often you don't need a lock at all: a unique constraint, or an atomic `UPDATE ... WHERE version = ?`, already gives you the guarantee without any explicit locking machinery — usually the cleaner fix for "the same webhook/request might get delivered twice."

---

### Testing & Debugging

**Consistently the weakest category** (3/10, 4/10, 4/10 across three sessions) — the same underlying scenario shape recurred each time: an intermittent, timing-dependent bug ("double-click submit twice fast," "occasionally in prod") that can't be reproduced by normal manual testing.

**The correct approach, step by step — the sequence itself is the thing being tested:**
1. **Reproduce reliably first.** A debugger, and honestly most tools, are close to useless until the bug can be triggered on demand. If the report includes a trigger ("double-click," "fast," "sometimes"), treat that as the literal repro recipe and recreate it mechanically — e.g. script two near-simultaneous requests (Postman runner, a small script, or a debugger breakpoint that pauses request A mid-flight while request B runs) rather than clicking a button quickly by hand, which rarely hits the exact race window.
2. **Be deliberate about tool choice for timing bugs.** A debugger can be the *wrong* first tool here — pausing execution at a breakpoint changes the timing of everything around it, which can make a race condition disappear entirely (a "Heisenbug"). Logging is safer as a first move because it observes without altering timing, and can run in production where the bug actually happens.
3. **Use structured logging, not `System.out.println` + redeploy.** `println` has no log levels (can't turn it up/down), isn't searchable/aggregable across instances, and requires a full redeploy for every adjustment. Prefer a real logger with levels (`DEBUG`/`TRACE` for detail that's normally off) and structured context — request ID, timestamp, user/order ID, thread/instance ID — so a specific failing request can be isolated out of a huge log dump. (The instinct to filter logs by a request/correlation ID was already present and correct.)
4. **Form an explicit hypothesis for *why* it's intermittent** before reaching for a tool — don't jump straight to "let's add logging" with no theory. Candidate hypotheses worth naming out loud for a "sometimes" bug: a non-idempotent endpoint being hit twice (double-submit/double-charge), an exception being silently swallowed inside a loop (job "skips" some items with no trace), or a job running on two instances/threads simultaneously.
5. **Binary-search debugging** once you have a hypothesis and can reproduce: narrow down *where* in the flow the bug is introduced by checking state at successively narrower points, rather than reading the whole flow top to bottom hoping to spot it.

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

**Also in the topic pool, worth reviewing:** unit vs integration tests (what each actually catches — the testing pyramid); arrange/act/assert structure and one-behavior-per-test naming; what makes a test flaky and why a flaky test is worse than no test (erodes trust in the whole suite); reading a stack trace to find the root cause among framework noise.

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

**Also worth reviewing:** never trusting client-side-only validation (hiding a button isn't security — always re-validate server-side); cookies vs. sessions vs. tokens for keeping login state.

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

**Also in the topic pool, worth reviewing:** vertical vs horizontal scaling and what breaks specifically when you go from one instance to two (in-memory session state, in-memory counters, anything not externalized); why a load balancer needs sessions to be either sticky or stored externally (Redis/DB) once there's more than one instance.

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

**Also in the topic pool, worth reviewing:** merge vs rebase at a basic level; what a reviewer is actually looking for in a PR, and how to respond to review feedback without getting defensive.

---

## 4. Priority List Before the Real Interview

Ranked by (a) how wrong the current understanding is, and (b) how likely it is to come up:

1. **`@Transactional` semantics** — rollback only on unchecked exceptions/`Error` by default; does not coordinate across separate transactions/requests. (Confidently wrong twice — highest priority.)
2. **Test-driving a bug fix** — before fixing any reported bug, write a new test that encodes it and confirm it fails first; don't rely on "check the existing tests." (08-01 — weakest score of that session, needed two hints and still didn't reach it independently; ties directly into Testing & Debugging being the weakest category overall.)
3. **Merge conflict resolution** — resolving means combining both changes into one correct version, not discussing with a teammate to pick one side wholesale. (07-29 — a fresh misconception on a very common daily task.)
4. **Mocking anti-patterns (over-mocking)** — recognize when every collaborator is mocked and the assertion just checks the stub's own return value; know to verify orchestration (`verify()`) or stop mocking everything. (07-29 — weakest score of that session, needed the most support.)
5. **Telling apart the concurrency problem shapes** and their distinct fixes: in-process race → atomics/locks; **cross-instance duplicate processing → distributed lock or idempotency, not `synchronized`** (which is JVM-local only — added 08-01, partially reasoned but the mechanism wasn't yet explicit); cross-transaction lost update → optimistic/pessimistic locking; duplicate-insert race → `UNIQUE` constraint (no locking needed).
6. **Sync vs async / message queue decoupling** — a non-blocking `async/await` call is still a direct call coupled to the downstream service's availability; a message queue is a stronger, different kind of decoupling. (07-29.)
7. **Debugging methodology for intermittent bugs**: reproduce first, understand why a debugger can distort timing bugs, use structured logging with correlation IDs — weakest category by average score across sessions.
8. **Request lifecycle fundamentals** (DNS → TCP → TLS → HTTP) — scored 2/10 with zero partial credit, a very standard interview question.
9. **Precise design-pattern definitions** (Facade vs. God class; Strategy vs. plain polymorphism) — the *judgment* of over-engineering is already solid; the *vocabulary* is not.
10. **`hashCode`/`equals` contract mechanism** — the bucket-index lookup in `HashSet`/`HashMap`; conclusion was right after a hint, mechanism needs to be stated unprompted. (07-29.)
11. **Maven/Gradle dependency conflict diagnostics** (`dependency:tree`, nearest-wins) — a concrete, practical gap.
12. **Idempotency terminology** — use the word explicitly and know the PUT-safe / POST-risky default.
13. **SQL injection fix precision** — know *why* `PreparedStatement` works (structure/data sent separately) and that validation isn't a substitute for it; the vulnerability and a concrete attack were already identified correctly. (07-29 — lower priority, mostly a polish item.)
14. **Environment/secrets config across dev/staging/prod** — know Spring profiles for non-secret config and platform-injected secrets for real ones, rather than hand-distributing per-environment files. (08-01 — lower priority, decent instinct already, just needs the specific tools named.)
15. **Naming polish**: state the **leftmost-prefix rule** explicitly for composite indexes, and **lazy loading** explicitly as N+1's root cause — both were reasoned correctly but left unnamed. (08-01 — lowest priority, the underlying reasoning was already right.)

## 5. Consistent Strengths (keep doing these)

- Layered architecture (controller/service/repository) reasoning.
- Tracing concurrency bugs step-by-step (thread interleaving) rather than reciting definitions — this produced the two best scores across all sessions.
- Good YAGNI/over-engineering instincts on design patterns.
- Willingness to reason out loud, ask clarifying questions, and give a best guess under uncertainty rather than freezing (explicitly praised in multiple session summaries).
- Commit-hygiene instincts have measurably improved and stuck across sessions — evidence that repeated practice on a topic here is translating into consistent recall.
- Giving concrete, non-trivial attack/impact scenarios instead of generic answers — e.g. the SQL injection answer named a specific `JOIN`-based data-exfiltration exploit rather than just "it's insecure" (07-29).
- Recognizing well-known mechanisms fast and correctly on the first try when the pattern is familiar — N+1 and composite-index leftmost-prefix behavior (both 08-01) were identified and reasoned correctly from first principles, no hint needed.
- Volunteering practical, real-world fixes unprompted even when the core mechanism explanation is incomplete — e.g. bringing up idempotency as a fix direction for cross-instance duplicate processing before the `synchronized`-is-JVM-local mechanism was fully explained (08-01, Q1).
