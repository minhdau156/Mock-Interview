# Backend Interview Knowledge Reference — Sessions 2026-08-12, 2026-08-17, 2026-08-19, 2026-08-20, 2026-08-21, 2026-08-25, 2026-08-30, 2026-09-01 & 2026-09-02

Study reference from these mock interview sessions. It lists concepts by topic. Code examples show the mechanism.

---

## 1. Core Java & CS Fundamentals

**String immutability and the hidden O(n²) cost of `+=` in a loop.**
```java
public String buildCsv(List<Order> orders) {
    String csv = "";
    for (Order o : orders) {
        csv += o.getId() + "," + o.getTotal() + "\n";   // fine at 20 rows, a hot spot at 50,000
    }
    return csv;
}
```
`String` is immutable. `csv += ...` cannot mutate the existing object. The compiler rewrites it roughly as `csv = new StringBuilder(csv).append(...).toString()`. Each iteration allocates a brand-new `StringBuilder`. It copies the entire accumulated string into it, appends the new piece, and converts back to an immutable `String`. Iteration *i* copies `i` characters. Total copying grows quadratically: O(n²), not O(n). A loop that looks instant at 20 rows becomes the profiler's top hot spot at 50,000.

```java
// Fix: one StringBuilder, reused across every iteration — genuinely O(n)
public String buildCsv(List<Order> orders) {
    StringBuilder sb = new StringBuilder();
    for (Order o : orders) {
        sb.append(o.getId()).append(',').append(o.getTotal()).append('\n');
    }
    return sb.toString();
}
```
Reflex: `+=` that builds a `String` in a loop allocates a new object and recopies everything each iteration. Replace it with one `StringBuilder` declared outside the loop.

---

## 2. OOP, Design Patterns & Low-Level Design

**Single Responsibility Principle — "one reason to change," not "one noun."**
```java
// Violation: three unrelated reasons to change bundled into one class
@Service
public class UserService {
    public User createUser(UserRequest request) {
        User user = userRepository.save(toEntity(request));
        emailClient.send(user.getEmail(), "Welcome!", welcomeTemplate(user));   // reason #2: email changes
        return user;
    }
    public void recordLogin(User user) {
        analyticsRepository.save(new LoginEvent(user.getId(), Instant.now()));  // reason #3: analytics changes
    }
}
```
The class "feels like one responsibility" because everything touches a `User`. SRP is about *reasons to change*, not the noun. This class changes when signup validation rules change. It changes when marketing edits the welcome email copy. It changes when the analytics team adds a field to the login event. Three unrelated teams touch the same file. Concrete costs:
- A test of `createUser` validation must mock email delivery, although the test is unrelated to email.
- Reuse of "send welcome email" in another signup path (for example an admin bulk-import script) needs a duplicated call. Or it routes an unrelated flow through `createUser`.

```java
// Fixed: split by reason-to-change, then compose via injection
@Service
public class UserService {
    private final NotificationSender notificationSender;
    private final AnalyticsRecorder analyticsRecorder;

    public User createUser(UserRequest request) {
        User user = userRepository.save(toEntity(request));
        notificationSender.send(user.getEmail(), "Welcome!", welcomeTemplate(user));
        return user;
    }
    public void recordLogin(User user) {
        analyticsRecorder.recordLogin(user.getId());
    }
}
```

**Dependency Inversion — the other half of the same fix.**
```java
// Still coupled: UserService depends on a CONCRETE class
public class UserService {
    private final EmailService emailService;   // swapping to push notifications means editing UserService
}

// Inverted: UserService depends on an ABSTRACTION
public interface NotificationSender {
    void send(String to, String subject, String body);
}
public class EmailNotificationSender implements NotificationSender { ... }

public class UserService {
    private final NotificationSender notificationSender;   // any implementation can be injected
    public UserService(NotificationSender notificationSender) { this.notificationSender = notificationSender; }
}
```
Class extraction fixes SRP. It does not fix DIP. If `UserService` still takes a concrete `EmailService`, a later switch to push notifications or SMS means editing `UserService`. Inject the *interface type* (`NotificationSender`). Then the concrete implementation can change, or a second one can be added, without touching the consumer. In Spring, the concrete class has `@Service` and implements the interface. The consumer declares the interface type in its constructor. The container wires the concrete bean automatically.

**Premature abstraction — building `Strategy` + `Factory` before a second real implementation exists.**
```java
interface NotificationStrategy { void send(Order order); }
class EmailNotificationStrategy implements NotificationStrategy { ... }
class SmsNotificationStrategy implements NotificationStrategy { ... }   // built, but nothing ever passes "SMS"

class NotificationStrategyFactory {
    NotificationStrategy create(String type) {
        if (type.equals("EMAIL")) return new EmailNotificationStrategy();
        if (type.equals("SMS")) return new SmsNotificationStrategy();
        throw new IllegalArgumentException("Unknown type: " + type);
    }
}
```
The app sends only email. There is no SMS feature, config, or ticket. The interface-plus-factory scaffolding serves a variation that does not exist yet. It only *might* exist someday. It costs something now: an extra interface, an extra class, and a factory lookup to trace. Nothing uses the flexibility.
```java
// Simpler, and equally correct until a second real implementation actually needs to ship
class EmailSender {
    void send(Order order) { /* send email */ }
}
```
The signal is not the class count. The signal is a **second real, active implementation** that exists or is committed, scheduled work. "We might want this later" is not a signal. Add the `Strategy`/`Factory` split when the second implementation must ship. Separately, a branch such as `SmsNotificationStrategy` that nothing calls is dead code. Flag it for removal, whatever the abstraction decision.

**A mutable instance field on a singleton `@Service` is shared state across every concurrent request. Locking it fixes the wrong problem.** (08-25, Q3, scored 4/10)
```java
@Service
public class ReportGenerator {
    private int rowCount;   // one field, shared by every concurrent caller

    public Report generate(List<Row> rows) {
        rowCount = rows.size();
        Report report = new Report();
        // ... processing that reads rowCount a few lines later ...
        report.setSummary("Processed " + rowCount + " rows");
        return report;
    }
}
```
`@Service` beans are singletons. One instance serves every request. `rowCount` is not a value to combine safely across threads (like a running total). It is per-request data. It must not live on a shared object. The instinctive fix (`synchronized`, `AtomicInteger`, locking the field) targets the wrong bug. Even with perfect synchronization, Thread A can set `rowCount = 5`. Before A reads it back, Thread B (after A releases any lock) sets `rowCount = 3`. A now reads `3`. Locking makes each single read or write atomic. It does not stop *interleaving* between one request's write and its own later read. Both requests race over one variable that holds two logically separate values.
```java
// Fix: no field at all — a local variable lives on each thread's own stack frame
public Report generate(List<Row> rows) {
    int rowCount = rows.size();
    ...
}
```
General principle: a singleton `@Service`/`@Component` must be **stateless**. Per-call data belongs in parameters, locals, or the return value. Never put it in an instance field. Prototype scope is *not* a reliable fix either. Spring usually resolves a bean's dependencies once, when the *injecting* bean (often a singleton) is constructed. A prototype-scoped `ReportGenerator` injected into a singleton controller is still reused as one instance in practice. A fresh instance per call needs a scoped proxy or `ObjectProvider`.

**LLD exercise (vending machine) — inheritance for data variation is the same over-engineering trap as an unnecessary interface. "Inject" is not the same mechanism as "look up by string."** (2026-09-02, Q3, scored 6/10 — fully self-directed, no hints needed)
```java
// Over-engineered: Pepsi/Coca/Tea don't BEHAVE differently, they just hold different DATA
class Drink { Long id; String type; int quantity; BigDecimal price; String code; }
class Pepsi extends Drink { ... }
class Coca extends Drink { ... }
class Tea extends Drink { ... }
```
The candidate correctly used an interface where behavior varies (`PaymentStrategy` with `CashPayment`/`CardPayment`). Validating and collecting money really differs by method. But the candidate then built a class hierarchy for `Drink` flavors that differ only in name, price, and code. They have no behavioral difference. This is the same principle as the Strategy/Factory premature-abstraction case, from the opposite direction. There, an interface served a variation that did not exist yet. Here, a *class hierarchy* serves a variation that was never behavioral. Correct model: one concrete `Drink` class, with multiple instances that hold different data.
```java
// What was described as "inject the PaymentStrategy" was actually two different mechanisms conflated:
public PaymentStrategy getDrink(String code, String paymentMethod) {
    PaymentStrategy strategy = createStrategy(paymentMethod);   // <- this is a FACTORY: string -> instance
    ...
}
// True injection: the caller already hands over a built instance, no branching on a string at all
public class DrinkHandler {
    private final PaymentStrategy strategy;   // <- THIS is injection
    public DrinkHandler(PaymentStrategy strategy) { this.strategy = strategy; }
}
```
Mapping a runtime string (`"CARD"`) to the right `PaymentStrategy` implementation is a **Factory**'s job. It is not "injection". Injection means the dependency arrives already built from the caller (for example Spring wiring a bean). No string-based lookup is involved. One design can use both (a Strategy interface plus a Factory that picks the implementation). They answer different questions. Do not describe them interchangeably.

Also note: the prompt explicitly asked for money/change state ("the variable that get the money and the variable hold the money") and for stock decrement and out-of-stock detection. The candidate never addressed them. Check a design answer against every stated requirement before calling it complete.

---

## 3. Databases: SQL, Indexing & Data Modeling

**A single FK column models one-to-many. Belongs-to-*multiple* needs a junction table, not more columns.**
```sql
-- Original: a product can have exactly one category
CREATE TABLE products (
  id BIGINT PRIMARY KEY,
  name VARCHAR(255),
  category_id BIGINT REFERENCES categories(id)
);

-- The instinct to resist: bolting on more columns per new requirement
-- category_id_2, category_id_3, ... — caps the relationship at a fixed number forever,
-- and "find all products in category X" becomes an OR across every column.
```
When a requirement says "a product belongs to *multiple* categories," the relationship is many-to-many. A single FK column on either side cannot express it. No fixed column count is ever enough. The standard fix is a **junction table** (also called a join table). Its only job is to record which pairs are linked.
```sql
CREATE TABLE product_categories (
  product_id BIGINT REFERENCES products(id),
  category_id BIGINT REFERENCES categories(id),
  PRIMARY KEY (product_id, category_id)
);
-- category_id comes OFF the products table entirely — categories now live only here
```
The two FK columns together form a **composite primary key**. It has two jobs. It identifies the row. It also guarantees that the same product cannot join the same category twice, because a repeated pair violates the PK. The query "all products in category X" also gets simpler: one join against `product_categories`, not an `OR` across a growing list of columns.

**A delimited list in one column violates First Normal Form. The fix is a child table. The "extra" foreign key rows are not duplication.** (08-25, Q2, scored 4/10 — needed two hints, one lingering misconception)
```sql
-- Original: two phone numbers crammed into one column
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    phone_numbers VARCHAR(255)  -- "555-1234,555-5678"
);

SELECT * FROM customers WHERE phone_numbers LIKE '%555-1234%';  -- "works," but can't use an index
```
1NF requires each column value to be **atomic**: one value, not a delimited list. Two numbers in one cell force every search into a wildcard scan. `LIKE '%555-1234%'` (leading `%`) cannot use a B-tree index, whatever the column shape. The index cannot binary-search for a value that can start anywhere in the string.
```sql
-- Fixed: one phone number per row
CREATE TABLE customer_phone_numbers (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    phone_number VARCHAR(20)
);
CREATE INDEX idx_phone_number ON customer_phone_numbers (phone_number);

SELECT * FROM customer_phone_numbers WHERE phone_number = '555-1234';  -- plain equality, uses the index
```
Misconception to correct: repeating `customer_id` as a foreign key across several phone rows is **not** harmful duplication. It is how a one-to-many relationship looks. The accepted trade-off: a `JOIN` is needed when a customer's name and phone numbers are needed together. In return you get correct indexing, an unbounded phone count, and no wildcard scans. Reflex: a column that must hold "a comma-separated list of X" signals a child table with one X per row. Do not use a wider `VARCHAR`.

**Sequential IDs are guessable. The fix (UUID) has real index and insert costs. A middle ground usually exists.** (2026-09-01, Q3, scored 3/10 — heavy hinting, asked for the answer outright twice)
```
GET /orders/1042   -- watching this endpoint over time reveals order volume/growth rate to anyone
```
An auto-increment primary key in a URL leaks more than an ID. The sequence is enumerable (walk `/orders/1`, `/orders/2`, ...). It also reveals a business metric: the rate of order creation, visible from the climbing numbers. A UUID (especially random v4) removes this. Nobody can guess a real order's ID from a known one.
```sql
-- BIGINT: 8 bytes, and every new value is greater than the last —
-- inserts always land at the rightmost edge of the B-tree (cheap, no reshuffling)
id BIGINT AUTO_INCREMENT PRIMARY KEY

-- UUID: 16 bytes, and (for v4) uniformly random —
-- inserts land at unpredictable points throughout the tree
id UUID PRIMARY KEY DEFAULT gen_random_uuid()
```
Switching the primary key to UUID is not free. It doubles the key storage, and the storage of every foreign key and index that references it. A random UUID insert pattern causes B-tree page splits and index fragmentation, because new rows no longer append at the end like sequential integers. That is the real mechanism, more precise than "lookups get slower." Time-ordered variants (UUIDv7, ULID) keep unguessability and keep enough ordering to avoid this fragmentation. Lookup of a row by a known ID is still roughly `O(log n)` either way. The cost sits in writes and index size, not reads.

```sql
-- Middle ground: the internal primary key and the publicly exposed identifier don't have to be the same column
CREATE TABLE orders (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,                        -- internal: fast joins, compact indexes, never exposed
  public_id UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),     -- external: what the API/URLs actually use
  ...
);
-- GET /orders/{public_id} -- e.g. /orders/3f29b1e0-...  -- nothing sequential is ever exposed
```
Internally, every join and foreign key uses the cheap, compact `BIGINT`. Externally, the API hands out only the opaque `public_id`. Nothing guessable leaves the system. Stripe uses the same pattern: public IDs like `ch_1a2b3c...` are not the internal database primary keys. Reflex: for the requirement "don't let this ID be guessable," check whether the *primary key* must change, or only what the API boundary exposes. The two are separable.

**A single `status` column holds only the current value. `UPDATE` destroys the transition history, not just "some detail."** (2026-09-02, Q2, scored 4/10 — needed the question broken into parts and three hints to reach the root cause)
```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    status VARCHAR(20),      -- 'PENDING', 'PAID', 'SHIPPED', 'DELIVERED', 'CANCELLED'
    updated_at TIMESTAMP
);

UPDATE orders SET status = 'PAID', updated_at = '2026-08-30 10:00' WHERE id = 42;
-- three days later:
UPDATE orders SET status = 'SHIPPED', updated_at = '2026-09-02 09:00' WHERE id = 42;
-- the row now has no trace that it was ever 'PAID', or when — that value was OVERWRITTEN, not archived
```
"When did it move to PAID, and how long did it sit there before shipping" cannot be answered here. The cause is structural, not a query problem. A row has exactly one `status` slot and one `updated_at` slot. Every `UPDATE` overwrites them. The previous value has nowhere to persist. Changing the column from `VARCHAR` to `ENUM` changes nothing. An `ENUM` column is overwritten by `UPDATE` in the same way. `ENUM` controls which strings are *valid*, not whether old values survive.
```sql
-- Fix: an insert-only history table — a transition becomes a new row, not an overwrite
CREATE TABLE order_status_history (
    id BIGINT PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id),
    status VARCHAR(20),
    changed_at TIMESTAMP
);
-- "how long did it sit in PAID" is now just: the next row's changed_at minus this row's changed_at
```
The real cost of this fix is not storage or indexing. A bigger table with an index on `order_id` is cheap. The cost is **write amplification and read complexity**. Most real designs also keep `orders.status` as a denormalized column for fast current-state reads. Then every transition needs *two* writes kept in sync: update `orders.status` **and** insert a history row. This usually happens in one service method, or in a DB trigger. Drop the denormalized column and reads get more expensive: the current status needs `SELECT status FROM order_status_history WHERE order_id = ? ORDER BY changed_at DESC LIMIT 1` instead of a single-column read. Reflex: for a requirement "show every past value/timestamp of X," check whether X is a column that gets `UPDATE`d in place. If so, the fix is almost always an insert-only child table. Do not use a wider column or extra metadata in the existing one.

---

## 4. Spring Boot & JPA/Hibernate

**`cascade = ALL` + `orphanRemoval = true` turns a plain collection mutation into a real `DELETE`.**
```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}

// elsewhere, in code that only intends to build a filtered display list:
order.getItems().remove(item);   // no explicit delete call anywhere in this code path
```
`orphanRemoval = true` tells Hibernate: "if an `OrderItem` is removed from this collection, it has no parent. Delete it from the database." `order.getItems()` returns the real, Hibernate-managed collection, not a copy. `.remove()` on it mutates the tracked association. Hibernate dirty-checking notices the change at flush time (end of the transaction). It issues a genuine `DELETE`, exactly as if `repository.delete(item)` had been called. The danger is intent mismatch. Code that only meant to filter a collection for *display* cannot signal "this is not a real removal." Any mutation on a managed collection inside an open transaction is a real write, whatever the intent.

**Misconception to name explicitly: a field initializer does not protect a loaded entity.**
```java
private List<OrderItem> items = new ArrayList<>();
```
This initializer runs only when you construct a new, transient instance with `new Order()`. When Hibernate loads an *existing* row, it replaces the field with its own managed collection wrapper (a `PersistentBag`) during hydration. The initializer's value is discarded. It is not merged with the real data. It does not protect it.

Fix: never call mutating collection methods (`remove`/`add`/`clear`) on a live, managed entity collection from read-only or reporting code. Copy it first (`order.getItems().stream().filter(...).toList()`). Use `orphanRemoval = true` only when "removed from the parent's collection" and "should be deleted" are the same business rule. Do not use it as a default bundled with `cascade = ALL`.

**`LazyInitializationException` — the session/transaction closes the instant the repository call returns, not when the entity is later used.** (08-25, Q1, scored 4/10 — flagged as an unintentional repeat of 08-05's Q1; see `KNOWLEDGE_SUMMARY.md` for the fuller writeup and the two standard fixes)
```java
@GetMapping("/customers/{id}")
public Customer getCustomer(@PathVariable Long id) {
    return customerRepository.findById(id).orElseThrow();   // no @Transactional anywhere
}
// Customer.orders is @OneToMany(fetch = FetchType.LAZY)

log.info("orders count: {}", customer.getOrders().size());   // throws: no Session
```
The method declares no `@Transactional`. A Spring Data repository method such as `findById` is transactional only for the duration of its own call. When it returns, the transaction commits and the session closes. `customer` is now **detached**. The log line is not the cause. It is only the first place that touches the lazy `orders` collection. Correct fixes:
- Wrap the logic in a `@Transactional` service method that returns a DTO (not the entity) to the controller.
- Or use a repository query with `JOIN FETCH` to load the association in the same query.
- Or project straight to a DTO in the repository layer.

Blanket eager fetching is not the fix. It loads the association on every fetch everywhere, including call sites that never need it.

**`@Value` vs. `@ConfigurationProperties` — fail-fast vs. silent lenient binding on a missing key.** (2026-09-02, Q1, scored 5/10 — needed two hints)
```java
@Component
public class PaymentGatewayClient {
    @Value("${payment.stripe.api-key}")
    private String apiKey;
    @Value("${payment.stripe.timeout-ms}")
    private int timeoutMs;
    // ... 6 more @Value fields for related Stripe settings
}
```
Eight separate `@Value` fields scattered across one class is a code-review smell. Group related settings into one `@ConfigurationProperties` class. It is more type-safe and unit-testable without a Spring context. It also supports nested and complex types that `@Value` cannot bind directly. The sharper difference appears when `payment.stripe.timeout-ms` is **missing** from `application.yml`:
```java
// @Value: fails LOUDLY at startup — Spring can't resolve the placeholder text itself
// throws: IllegalArgumentException: Could not resolve placeholder 'payment.stripe.timeout-ms'
@Value("${payment.stripe.timeout-ms}")           private int timeoutMs;
@Value("${payment.stripe.timeout-ms:5000}")      private int timeoutMs; // fallback avoids the throw

// @ConfigurationProperties: fails SILENTLY — relaxed/lenient binding just leaves
// the field at its Java default. No exception, no log line, app starts normally.
@ConfigurationProperties(prefix = "payment.stripe")
public class StripeProperties {
    private int timeoutMs;   // missing key in application.yml -> stays 0, quietly
}
```
Treating the silence of `@ConfigurationProperties` as a plus ("it can work well") is backwards. A silently defaulted `timeoutMs = 0` can make every Stripe call time out instantly in production. Nothing at startup flags it. The fail-fast behavior of `@Value` is the safer default in isolation. To keep the other benefits of `@ConfigurationProperties` (grouping, type-safety, testability) and close this gap, add `@Validated` on the properties class. Add `@NotNull`/`@Min(1)` on the fields. A missing or invalid value then fails at startup instead of silently. Reflex: when you compare two config-binding approaches, first ask "does this fail loudly or silently?" Silent failure is nearly always the more dangerous property.

---

## 5. REST & API Design

**A schema-accurate API spec can still be unusable. Types give the shape, not the exact contract.**
```yaml
# Technically complete and correct — every field, type, and optionality is right
CreateOrderRequest:
  type: object
  properties:
    customerId: { type: string }
    items: { type: array, items: { $ref: '#/OrderItem' } }
    discountCode: { type: string }
  required: [customerId, items]
```
An integrating team can read this spec and still get it wrong, and create real support tickets:
- Is `discountCode` omitted, or sent as `""`? The spec does not say whether they behave differently.
- What format does a date field expect: `2026-08-12`, an ISO-8601 timestamp, or a Unix epoch?
- What does a *valid* `OrderItem` look like inside `items`? Is there a minimum count? What if one item carries its own discount?

None of this is a documentation *error*. The schema is accurate. It is a documentation *gap*. A type system describes shape, not the concrete values that satisfy real use cases.

**Unbounded list endpoints eventually break. `LIMIT` alone hides data instead of fixing the problem.**
```sql
-- The "fix" that isn't one: stops the timeout, but nothing lets a client ever ask for row 101+
SELECT * FROM orders WHERE customer_id = ? LIMIT 100;
```
This bounds the response. But the API has no `page`/`offset` or cursor parameter. The rest of that customer's 40,000 rows are no longer slow to reach. They are unreachable. It also has a second, subtler bug. Without an `ORDER BY`, SQL does not guarantee *which* 100 rows return. "The first 100" can differ between identical calls.

```sql
-- Offset pagination: simple, but cost grows with depth
SELECT * FROM orders WHERE customer_id = ? ORDER BY id LIMIT 100 OFFSET 39900;
-- the DB scans and discards 39,900 rows before it can return the 100 you actually asked for

-- Cursor (keyset) pagination: flat cost regardless of depth
SELECT * FROM orders WHERE customer_id = ? AND id > :lastSeenId ORDER BY id LIMIT 100;
-- an index seek straight to the row after the cursor — nothing scanned and thrown away
```
The cost of `OFFSET n` grows with `n`. The database must walk through and discard every skipped row first. A cursor on an indexed, stable column (`id`, or `created_at` plus an `id` tiebreaker) seeks directly to where the last page ended. Performance stays flat at page 2 or page 400. The API contract becomes `GET /orders?cursor=<lastId>&size=50`. It returns the page plus a `nextCursor` for the next call. Reflex: an endpoint that returns "all rows matching X" with no `page`/`size` or `cursor` parameter will eventually break, when one owner's data outgrows a single response. Add pagination before that happens, not after the ticket arrives.

**The fix is a worked example, not more prose.**
```yaml
CreateOrderRequest:
  # ...schema unchanged...
  example:
    customerId: "cust_492"
    items:
      - { sku: "SKU-100", quantity: 2 }
      - { sku: "SKU-204", quantity: 1, discountCode: "SUMMER10" }
    discountCode: null
```
A concrete `example`/`examples` block in the OpenAPI spec answers all the questions above at a glance. The integrating developer can copy it, run it, and see it work. Otherwise they infer from field types, guess, and debug against 400 responses. Reflex: a bug report says "the docs are wrong," but the schema checks out field by field. The fix is almost never *more explanatory text*. It is a runnable example that covers realistic cases: a plain request, one with an optional field populated, and an edge case such as a minimal request.

---

## 6. Concurrency & Locking

**`synchronized` protects against overlapping threads at the same instant. It does nothing for a retried request that arrives later.**
```java
// Doesn't do what it looks like it does, for this problem:
@PostMapping("/webhooks/payment-confirmed")
public synchronized ResponseEntity<Void> handlePaymentWebhook(@RequestBody PaymentEvent event) {
    orderService.markPaid(event.getOrderId());
    return ResponseEntity.ok().build();
}
```
A payment provider's webhook retry is not a second thread racing the first at the same moment. It is a new, separate HTTP request. It arrives seconds or minutes after the first one finished and returned. `synchronized` blocks a thread from entering a block only while another thread is *currently inside it*. Once the first request completes, nothing remains to block against. The bug this should catch, the same payment event processed twice, passes straight through.

**When a duplicate is a matter of identity, not timing, the fix is a uniqueness guarantee, not a lock.**
```sql
CREATE TABLE processed_webhook_events (
  event_id VARCHAR(255) PRIMARY KEY
);
```
```java
public void handlePaymentWebhook(PaymentEvent event) {
    try {
        processedEventRepository.insert(event.getEventId());   // fails if already present
    } catch (DuplicateKeyException e) {
        return;   // already processed — safely ignore and return 200 again
    }
    orderService.markPaid(event.getOrderId());
}
```
The webhook payload carries a unique event ID so a handler can atomically check and record "have I seen this before?" A `UNIQUE`/primary-key constraint on `event_id` makes that check atomic at the database level. It works however many requests arrive, and however far apart in time. It needs no `synchronized`, no distributed lock, and no extra infrastructure. Reflex: separate "two requests at the *same instant*" (a concurrency problem, needs a lock) from "the *same* request arriving again later" (an identity/idempotency problem, needs a uniqueness guarantee). They look similar but need different tools.

**A single-JVM lock (`synchronized`) does not coordinate across instances. Cross-instance mutual exclusion needs an external, atomic check-and-set.**
```java
// Looks like it should prevent triple-execution — doesn't, once there's more than one instance
@Scheduled(cron = "0 0 2 * * *")
public synchronized void sendAbandonedCartReminders() {
    cartService.emailAbandonedCarts();
}
```
Scaling from one instance to three does not multiply threads inside one JVM. It creates three separate JVMs. Each has its own thread pool and its own independent `synchronized` lock. All three fire their own 2 AM cron trigger. Each lock blocks only *other threads in the same JVM*. `synchronized` does not know the other two instances exist. The job still runs three times.

```java
// Redis SET ... NX PX: one atomic command, only one instance can win it
Boolean acquired = redisTemplate.opsForValue()
    .setIfAbsent("lock:abandoned-cart-job", instanceId, Duration.ofMinutes(10));

if (Boolean.TRUE.equals(acquired)) {
    try {
        cartService.emailAbandonedCarts();
    } finally {
        redisTemplate.delete("lock:abandoned-cart-job");
    }
}
// other two instances: setIfAbsent returns false, they skip the job entirely
```
`SET key value NX PX <ttl>` succeeds only if the key does not exist. The check-and-set is one atomic operation. No window exists where two instances both pass a "does it exist?" check before either writes. The `PX <ttl>` expiry is not optional. Without it, an instance that acquires the lock and crashes mid-job leaves the key set forever. The job then never runs again on any instance. With a TTL, the lock expires even if the holder never releases it. The next scheduled run can then acquire it. Reflex: when `synchronized` is proposed as the fix for "duplicate work across multiple instances/services," ask "which JVM does this lock live in?" A lock scoped to one process cannot coordinate processes that do not share its memory.

---

## 7. Testing & Debugging

**Read the exception message before the frames. It often already names the null and the failed call.**
```
java.lang.NullPointerException: Cannot invoke "String.trim()" because the return value of "com.acme.orders.OrderService.getCustomerEmail(Order)" is null
	at com.acme.orders.NotificationSender.send(NotificationSender.java:42)
	at com.acme.orders.OrderController.checkout(OrderController.java:87)
	at org.springframework.web.method.support.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:150)
	... 40 more lines of Spring internals ...
```
Modern Java "helpful NullPointerException" messages state what was null and which operation failed on it. Here, `getCustomerEmail(Order)` returned null, and `.trim()` was called on that result. That sentence alone often narrows the search before you read a single frame.

**Skip framework frames on purpose. Stop at the top-most frame that is your own code.**
The dozens of `org.springframework.*` lines below the crash are Spring's dispatch machinery calling into the controller. They are not the bug. Skip every frame whose package is not yours. Stop at the first one that is. Here that is `NotificationSender.java:42`, called from `OrderController.java:87`. This habit scales. You read a 5-line trace and a 50-line trace the same way. The second has more noise to skip.

**A null-guard at the crash site fixes the crash, not the bug.**
```java
// Stops the NPE, but doesn't explain anything
if (email != null) {
    email.trim();
}
```
This prevents the exception. The real question is *why* `getCustomerEmail(Order)` returned null in production. Possible causes: a guest checkout with no email on file, a migration that left old rows without one, or an upstream bug in how the order was built. A guard at the call site is a reasonable safety net. If you stop there, the same root cause can resurface anywhere else that touches a null email. In production, also tie the trace back to the actual order or request that triggered it (via logs or the order ID). You usually cannot re-run it to reproduce.

**To find which step failed in a multi-call chain, use what is already captured, or a trace, before you scatter new print statements.**
```java
// checkout chains five calls; a generic catch-all hides which one actually failed
try {
    validateCart(cart);
    applyPricing(cart);
    reserveInventory(cart);
    chargePayment(cart);
    sendConfirmation(cart);
} catch (Exception e) {
    log.error("Checkout failed", e);           // logs *something*, but the client only sees:
    return ResponseEntity.status(500).body("Checkout failed, please try again");
}
```
"Add a `System.out.println` after every line across all five methods and redeploy" turns every guess into a full redeploy cycle. It also tells you only about the *next* failure, not this one. Two better moves, in order:
1. **Check what is already captured.** The generic `catch (Exception e)` above still logs the real, specific exception (`log.error("Checkout failed", e)`), although the client sees only a generic message. The answer is very likely already in the logs, searchable by request ID, with no new deployment.
2. **For a live failure that is hard to reproduce locally, use tracing instead of prints.** A distributed tracing tool (for example Tempo, viewed through Grafana, correlated with logs in Loki) records a span for each of the five calls, with timing and status. You do not guess which of five methods to instrument. You look at the trace and see which span failed or where the chain stopped.

Both techniques rely on **binary-search debugging**. Narrow the search space by checking a midpoint ("did `reserveInventory` complete? yes → the bug is in the last two steps"). Do not instrument the whole chain at once and hope the output is obvious. Reflex: before you add new logging or print statements to hunt a bug, ask "is this already captured somewhere I have not looked (existing logs, an existing trace)?"

**Log levels form a threshold hierarchy. Enabling `DEBUG` does not replace `INFO`/`WARN`/`ERROR`. It captures all of them plus everything below.** (2026-09-01, Q1, scored 4/10 — needed the level definitions handed over before attempting the scenario)
```
ERROR  -- something failed and needs attention (payment call failed after retries, DB save failed)
WARN   -- recovered from something unexpected (cache miss fell back to DB, deprecated API called)
INFO   -- small number of meaningful lifecycle/business events (order placed, app started)
DEBUG  -- fine-grained internals (computed discount = 0.1 for customer 42, entering method X)
```
Setting the root logger to `DEBUG` does not swap out the other levels. It lowers the threshold. Every `DEBUG` line *and* every `INFO`/`WARN`/`ERROR` line above it is emitted. This is why flipping a service to `DEBUG` during an incident, and forgetting to revert it, makes volume balloon. Normal traffic that produced a handful of `INFO` lines per request now also emits every fine-grained internal step, for every request, indefinitely.

**Fixing the level alone does not fix the process failure that let it linger, or the real complaint "can't find the right lines."** Three separate problems usually arrive bundled in such a scenario. Reverting the level is only the first:
1. **The level itself.** Revert to `INFO` (or the service's steady-state level) once the investigation that justified `DEBUG` ends.
2. **The process gap that let it linger for weeks.** The root cause is not "the level was wrong." A *temporary* diagnostic change had no mechanism to force it back. A follow-up reminder or ticket tied to any manual level change, or an auto-expiring override, stops "someone forgot" from persisting silently.
3. **Findability.** Lower volume helps. But the real fix for "took 40 minutes to find the relevant lines" is structured logging with a searchable correlation/request ID on every line. You can then filter one request's lines instantly, however noisy the overall stream is.

Also note: a disk-usage alert caused by log volume is a **retention/rotation** gap as much as a level gap. Even a correctly leveled service needs old logs to expire or rotate. Then a future legitimate volume spike (a traffic surge, a new feature) cannot fill the disk again. `DEBUG` is generally safer when scoped narrowly (one package/logger, or one request) than when flipped at the root logger for the whole service. That containment makes "forgot to revert" cost much less next time.

---

## 8. Web Security & Frontend Basics

**`useEffect` dependency arrays compare by reference, not by value. An inline object/array/function literal breaks this silently.**
```jsx
function ProfilePage({ userId }) {
  const [profile, setProfile] = useState(null);
  const options = { include: 'address' };   // a NEW object, every single render

  useEffect(() => {
    fetchProfile(userId, options).then(setProfile);
  }, [options]);   // linter says "add options" — doesn't say it's the wrong shape of value to depend on

  if (!profile) return <Spinner />;
  return <ProfileCard profile={profile} />;
}
```
React compares each dependency-array entry with `Object.is`. This is reference/identity equality, not deep or value comparison. `{ include: 'address' }` is structurally identical on every render. But it is a *different object in memory* each time the component function runs. React sees "changed" every time. The loop: render creates a new `options` → effect sees a new reference → runs `fetchProfile` → `.then(setProfile)` updates state → the state update triggers a re-render → a new `options` object is created → repeat forever.

```jsx
// Fix: give the dependency a stable reference across renders
const options = useMemo(() => ({ include: 'address' }), []);   // idiomatic — stable identity for a static value

// or, simplest of all when the value truly never changes: hoist it out of the component
const OPTIONS = { include: 'address' };
function ProfilePage({ userId }) { ... }
```
`useState` also "works" here. The initial value is constructed once, and the reference stays stable unless `setOptions` is called. But it is not the idiomatic tool. `useState` signals a value that changes over time, and this value does not. Reflex: when an object/array/function literal is defined in the component body and placed in a dependency array, ask "same reference every render, or a new one?"

**Conditional rendering of a UI element is UX, not authorization. The server must re-check every time, because the client can always be bypassed.**
```jsx
// Hides the button from a non-admin user in the UI — and only that
{user.role === 'admin' && (
  <button onClick={applyEmployeeDiscount}>Apply Employee Discount</button>
)}
```
```
# Bypasses the button, the component, and the role check entirely:
curl -X POST https://api.example.com/discounts/employee -H "Authorization: Bearer <any-valid-token>"
```
`{user.role === 'admin' && ...}` decides only whether *this* button renders in *this* browser tab. It has zero effect on whether the backend endpoint accepts a request. React, the component tree, and `user.role` do not run on the server. A non-admin user, or anyone with a valid token, can call the same endpoint directly with curl, Postman, or Network-tab replay. The server processes it exactly as if the button had been clicked. For the server, no button was ever involved.
```java
// The check has to be re-asserted here, independent of anything the client sent or hid
@PreAuthorize("hasRole('ADMIN')")
@PostMapping("/discounts/employee")
public ResponseEntity<Void> applyEmployeeDiscount(@AuthenticationPrincipal UserDetails user) {
    discountService.applyEmployeeDiscount(user);
    return ResponseEntity.ok().build();
}
```
`@PreAuthorize` checks the role carried in the authenticated principal. That role comes from a signature-verified JWT claim or a server-side session. The client cannot forge it, unlike a plain `user.role` value in browser-side JS state. Keep the frontend check: there is no reason to show a button an employee cannot use. But it is UX polish layered *on top of* server-side authorization, never a substitute. Reflex: for any sensitive action wired to a UI element, ask "what stops someone from hitting this endpoint directly, skipping the UI?" If the answer is "nothing," the button secured nothing.

---

## 9. System Design & Architecture Basics

**Caching is a staleness-vs-load trade-off. The acceptable staleness window depends on the cost of being wrong, not only on read volume.**
```
Admin updates price  ---->  DB price = $80
                                          \
Cache still holds     ---->  Cached price = $100   (TTL: 10 minutes)
                                          \
Customer checks out during that window  ---->  charged $100 for something that's actually $80
```
A cache in front of a hot read (`GET /products/{id}/price`) is the right instinct when the DB struggles under read load. But the chosen TTL decides how long a wrong value survives after the source of truth changes. A long TTL for a rarely changing field (product description, image URL) is a fine trade. Mild staleness costs nothing. The same TTL on price data means "for up to 10 minutes, any cached number might be wrong." If that number is charged, not just displayed, staleness stops being a UX nuisance. It becomes a financial and legal problem.

**A shorter TTL narrows the window. It does not close it. Invalidate-on-write is the complementary fix, not a substitute.**
```java
// Cache-aside with invalidate-on-write: the write path evicts the stale entry
// immediately, instead of waiting for it to expire.
public void updatePrice(Long productId, BigDecimal newPrice) {
    priceRepository.updatePrice(productId, newPrice);
    priceCache.evict(productId);   // next read is a guaranteed cache MISS -> fresh from DB
}
```
Even with a short TTL *and* invalidate-on-write, a race remains. A request already in flight when the price changes can read the old cached value just before eviction. For data where being wrong has a real cost, split the read path by purpose. Serve the *cached* value for display. Re-verify against the *authoritative* source at the exact moment the value is acted on.
```java
public OrderResult checkout(Order order) {
    // Do NOT trust the cached price here, however fresh it looks.
    BigDecimal authoritativePrice = priceRepository.findCurrentPrice(order.getProductId());
    return paymentGateway.charge(authoritativePrice);
}
```
Rule of thumb: cache freely for reads whose worst case is "the user saw a slightly stale number" (search results, product listings). Never let a cached value be the thing that is charged, allocated, or committed. Re-fetch from the source at that instant.

**A missing service layer is not a style nit. It has four concrete costs, each independent of the others.**
```java
// Business logic living directly in the controller, no OrderService anywhere
@PostMapping("/orders/{id}/apply-discount")
public ResponseEntity<Order> applyDiscount(@PathVariable Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    if (order.getCustomer().getLoyaltyPoints() > 1000) {
        order.setTotal(order.getTotal() * 0.9);
    }
    orderRepository.save(order);
    return ResponseEntity.ok(order);
}
```
A controller's job is to translate HTTP in and out: parse the request, call the service, map the result to a status code and body. Loyalty points and discount math are not part of that. When a business rule lives in the controller, the controller has two unrelated reasons to change: an HTTP contract change and a discount-rule change. Four concrete costs follow, not just "it is harder to read":
- **Testability.** To test "does a 1000+ point customer get 10% off," you need `MockMvc`, a fake `@PathVariable`, and a fake `HttpServletRequest`. That is full MVC machinery to test a calculation. A plain `OrderService` method is a one-line unit test with none of that.
- **Reusability.** A batch job, an admin tool, or another endpoint that later needs the same discount rule cannot reach it inside this one controller method. A service method is callable from anywhere.
- **Transaction boundaries.** `@Transactional` idiomatically belongs on the service method. As the logic grows (discount + inventory check + audit log), that annotation makes it one atomic unit. The controller is not the natural place for it.
- **Change isolation.** A pure HTTP-layer change (renaming the route, changing the response envelope) and a pure business-rule change (adjusting the loyalty threshold) touch the same method. Neither change is isolated from the other.

```java
// Fixed: controller only translates HTTP; OrderService owns the rule
@PostMapping("/orders/{id}/apply-discount")
public ResponseEntity<Order> applyDiscount(@PathVariable Long id) {
    return ResponseEntity.ok(orderService.applyLoyaltyDiscount(id));
}

@Service
public class OrderService {
    @Transactional
    public Order applyLoyaltyDiscount(Long id) {
        Order order = orderRepository.findById(id).orElseThrow();
        if (order.getCustomer().getLoyaltyPoints() > 1000) {
            order.setTotal(order.getTotal() * 0.9);
        }
        return orderRepository.save(order);
    }
}
```

---

## 10. Tooling: Git, Build & Environment

**Green CI proves the code satisfies the tests that already exist. It does not show that those are the right tests, or that the code is reviewable for reasons unrelated to passing.** (2026-09-01, Q2, scored 4/10 — correct core instinct, but no concrete example given despite being asked for one)
```
CI checks:             Does it compile? Do existing tests pass?
Still open questions:  Do the tests cover the cases that actually matter (edge cases, error paths)?
                        Is the code itself something a teammate could read/extend later?
                        Does it solve the problem that was actually asked for?
                        Does anything here need a check that no test was ever written to catch?
```
A PR can be fully green and still deserve pushback. Concrete examples that CI structurally cannot catch:
- **Untested edge cases and error paths.** The happy path is covered. A null input, an empty list, or a downstream failure has no test. A real bug there would ship silently.
- **Requirement mismatch.** The code is clean and fully tested but does not implement what the ticket asked for. CI knows only what the tests assert happened, not what should have happened.
- **Security gaps with no test.** Examples: a new endpoint without an authorization check, a hardcoded secret instead of one injected from config, unvalidated input. These pass every existing test, because nothing was written to catch them.
- **Architectural fit.** Business logic sits in a controller when the codebase convention is a service layer. Or new logic duplicates a utility that exists elsewhere. Both compile and pass tests.
- **Readability and maintainability.** Code that works but is hard to follow six months later has no test that can flag "this was confusing." It is still a legitimate reason to request changes.

Reflex: CI answers "does this work, according to the tests we already wrote." Review covers everything CI structurally cannot check. Start with "were these the right tests" and "is this what was asked for."
