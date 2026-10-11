# Backend Interview Knowledge Reference

Study reference from mock interview sessions. It contains concepts only, grouped by topic. Code examples show the mechanism.

---

## 1. Core Java & CS Fundamentals

**Checked vs unchecked exceptions. The class hierarchy decides, not intent.**
```
Throwable
├── Error                     (unchecked — JVM-level, e.g. OutOfMemoryError)
└── Exception
    ├── RuntimeException      (unchecked — NullPointerException, IllegalArgumentException, ...)
    └── everything else       (checked — IOException, SQLException, ...)
```
The compiler forces a method to `catch` or `throws` each checked exception. Checked exceptions represent *expected, recoverable* failures (a file is missing, a network call fails). The API tells each caller: "You must decide what to do here." `RuntimeException` subtypes are unchecked because they usually represent *programming bugs* that can occur on almost any line (null reference, bad cast). Requiring `throws NullPointerException` everywhere would add noise.

**Resource leaks. Use try-with-resources.**
```java
// Leaks the file handle if readLine() throws, and even on the happy path.
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
Reflex: a method opens a `Reader`/`Stream`/`Connection`/`Socket` with `new`. Ask: "Does this need try-with-resources?" The class must implement `AutoCloseable`. Nearly all I/O classes do.

**Big-O in practice.**
```java
// O(n × m) — 10,000 orders × 5,000 customers = 50,000,000 comparisons
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
Decision rule: use `Set<K>` to test existence only. Use `Map<K, V>` to get the object back.

**Date/time pitfalls.**
```java
// Wrong: LocalDateTime has no timezone/offset attached at all.
LocalDateTime sendAt = LocalDateTime.now().withHour(9).withMinute(0);

// Instant fixes "what exact moment is this" but is a single global moment,
// not "9am for this specific user."
Instant sendAt = Instant.now();

// Correct: derive from the user's zone, so "9am" means their 9am,
// then convert to Instant/UTC only for storage/scheduling.
ZoneId userZone = user.getZoneId();
ZonedDateTime local9am = ZonedDateTime.of(LocalDate.now(userZone), LocalTime.of(9, 0), userZone);
Instant sendAt = local9am.toInstant();
```
Rule: when `LocalDateTime.now()` touches anything cross-timezone, ask "Whose 9am is this?" If the answer depends on the reader, a `ZoneId` is required.

**`hashCode`/`equals` contract. Overriding only one silently breaks `HashSet`/`HashMap`.**
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
`HashSet.add()` does **not** compare the new object to every existing element with `.equals()`. It first calls `hashCode()` to compute a **bucket index** (`hash % numBuckets`). It then compares with `.equals()` only against elements in that *same* bucket. Two `Customer` instances with the same `customerId` but loaded as separate objects have different identity-based hash codes. They land in different buckets, so `.equals()` is never called. The contract: *"If two objects are equal via `.equals()`, they must return the same `hashCode()`."* Reflex: when you override `equals()`, generate `hashCode()` in the same action.

**Mutating a `HashMap` key after insertion. The bucket is fixed at `put()` time and never recomputed.**
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
`HashMap` computes `hashCode()` once at `put()` time to pick a bucket. The entry stays there. `get()` recomputes `hashCode()` and searches only the bucket that the *current* hash points to. `String` does not have this problem: it is immutable and caches its hash after the first call. Reflex: each field used by `equals()`/`hashCode()` in a `Map`/`Set` key class must stay effectively immutable while the object is a key.

**Integer caching / autoboxing. `==` "works" by accident for small values.**
```java
Integer a = 100;
Integer b = 100;
System.out.println(a == b);   // true

Integer c = 200;
Integer d = 200;
System.out.println(c == d);   // false
```
`Integer a = 100` autoboxes through `Integer.valueOf(int)`, not `new Integer(100)`. `valueOf()` keeps an internal cache (`IntegerCache`) for values **-128 to 127**. A value in that range returns the *same cached object* each time, so `==` (reference comparison) returns `true` by chance. Outside that range, `valueOf()` allocates a new object each call, so `==` correctly says "different objects", but the code did not mean to ask that. Fix: **always use `.equals()`** to compare wrapper types by value. Reflex: treat each `==` next to a wrapper type (`Integer`, `Long`, `Boolean`, `Character`) as suspect.

**Also worth knowing:**
- `HashMap` internals: collision handling (linked list, or red-black tree since Java 8 when a bucket is large enough). Resizing/rehashing on load factor.
- Binary search precondition: the collection must already be sorted.

---

## 2. OOP, Design Patterns & Low-Level Design

**YAGNI judgment.** Before you use a pattern, ask: *"What specifically will vary, and how often?"* If you cannot name a concrete future change, the pattern is premature. Strategy/Factory over 2-3 static, unchanging branches is usually overkill.

**Pattern definitions. Identify a pattern by the problem it solves, not its shape.**

| Pattern | Solves | NOT this |
|---|---|---|
| **Strategy** | Swap an *algorithm/behavior* at runtime through a common interface | "Interface + multiple implementers with a shared field". That is plain polymorphism |
| **Facade** | Simple unified entry point to a *complex, multi-class subsystem* (e.g. `CheckoutFacade` coordinating `InventoryService`, `PaymentGateway`, `NotificationService`) | "Putting everything into 1 class". That is the **God class** anti-pattern |
| **Factory** | Centralize *object creation*. Callers do not need to know which concrete class to instantiate | "A class with a `create()` method" with no real decision inside |
| **Singleton** | Guarantee exactly one shared instance (Spring beans are singleton-scoped by default, managed by the container) | Manual private constructor + static `getInstance()`. In Spring, let the container do this |
| **Observer** | Decouple "something happened" from "who reacts" (`ApplicationEvent`/`@EventListener` in Spring) | Direct method calls between unrelated classes |
| **Adapter/Decorator** | Wrap one interface to satisfy another, or add behavior without changing the wrapped object (`InputStream` wrappers like `BufferedInputStream`) | — |

**Strategy + Factory, concretely:**
```java
public interface NotificationSender {           // named for the verb it performs
    void send(String message, String recipient);
}
public class EmailSender implements NotificationSender { ... }
public class SmsSender implements NotificationSender { ... }

public class NotificationSenderFactory {
    private final Map<String, NotificationSender> senders;
    public NotificationSender get(String type) { return senders.get(type); }
}

notificationSenderFactory.get(type).send(message, recipient);   // no if/else at all
```
Naming rule: name a strategy interface for the **verb** it performs (`PayCalculator`, `NotificationSender`), not a noun.

**Composition over inheritance. When a subclass tree breaks down.**
```java
// Breaks down once ContractorEmployee needs "hourly-style pay" AND
// "a unique tax rule" — two independent axes that don't fit one subclass cleanly.
abstract class Employee { abstract double calculatePay(); }
class SalariedEmployee extends Employee { ... }
class HourlyEmployee extends Employee { ... }

// Composition: inject the varying behavior instead of subclassing further.
interface PayCalculator { double calculate(Employee e); }
class HourlyPayCalculator implements PayCalculator { ... }
class ContractorPayCalculator implements PayCalculator { ... }

class Employee {
    private final PayCalculator payCalculator;
    double calculatePay() { return payCalculator.calculate(this); }
}
```
The OOP pillar that does the work is **polymorphism**. The concrete `PayCalculator` runs at runtime through the interface. No if/else on employee type is needed. `Employee` can stay a base class for shared fields (name, id). An abstract class for shared state and an interface for varying behavior can coexist.

**Dependency Inversion Principle.** High-level modules must not depend on low-level modules directly. Both must depend on abstractions.
```java
// Violation: OrderService instantiates the concrete implementation itself
public class OrderService {
    public void checkout(Order order) {
        PaymentGateway gateway = new StripePaymentGateway();
        gateway.charge(order.getTotal());
    }
}

// Fixed: depend on the abstraction, inject the concrete implementation
public class OrderService {
    private final PaymentGateway gateway;   // interface type
    public OrderService(PaymentGateway gateway) { this.gateway = gateway; }
    public void checkout(Order order) { gateway.charge(order.getTotal()); }
}
```
The benefit is more than fewer code changes when the constructor changes. You can swap implementations (add PayPal later). You can unit-test business logic without a real payment API. In Spring: mark the concrete class `@Service` implementing the interface, inject the interface type through the constructor, and the container wires the concrete bean automatically.

**LLD process. Separate the two questions.** For each low-level design exercise (parking lot, vending machine, library system), write two lists before you sketch classes. List 1: *nouns with state* (what data exists, and whether it persists across requests or resets per transaction). List 2: *nouns with behavior*. Before you add an interface, name the concrete thing that differs between implementations. If no behavior varies (e.g. drink types that differ only in name/price), use one concrete class/record with many instances, not one interface per variant. Do not use "Factory" for inventory/repository work. Tracking and mutating stock state is not object creation.

**Tell, don't ask / anemic domain model.**
```java
// Anemic: Order is a passive data bag; OrderService reaches in and decides externally.
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

// Fixed: tell the object to do it to itself.
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
// Caller now just calls: order.ship();
```
Why it matters beyond style: the invariant ("ship only if paid and total is positive") lives in one place. It does not depend on each caller checking it correctly. Reflex: a service method computes a value only from one object's own getters, with no other collaborator. Ask: "Should that object do this to itself?"

---

## 3. Databases: SQL, Indexing & Data Modeling

**PRIMARY KEY vs UNIQUE.**
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,        -- guarantees id is unique — nothing else
  email VARCHAR(255) UNIQUE,    -- this is what actually stops duplicate emails
  name VARCHAR(255)
);
```
`PRIMARY KEY` protects only the key column. An app-level "check if it exists, then insert" is not a real guarantee, because it is not atomic.

**The classic duplicate-insert race. Only a DB constraint closes it:**
```
Time  Request A                       Request B
t0    SELECT ... WHERE email=X        
t1                                    SELECT ... WHERE email=X
t2    (no rows found)
t3                                    (no rows found)
t4    INSERT (email=X)
t5                                    INSERT (email=X)   -- duplicate!
```
Both requests read "not found" before either writes. Only a **DB-level `UNIQUE` constraint** closes this gap, because the DB enforces it atomically at insert time. The DB rejects the second `INSERT`, and the app catches the constraint-violation exception. No row exists to lock, so pessimistic locking does not apply. Pessimistic locking protects *existing* rows, not the absence of a row.

**Index mechanics and how to defeat an index.**
```sql
-- Index on email is USED
SELECT * FROM users WHERE email = 'someone@example.com';

-- Index on email is DEFEATED — the DB must compute UPPER(email) for every
-- row before comparing, so it can't use the b-tree ordering built on raw values.
SELECT * FROM users WHERE UPPER(email) = UPPER('someone@example.com');
```
A B-tree index keeps *raw* column values in sorted order, so lookups and range scans cost `O(log n)`. If you wrap the column in a function (`UPPER()`, `YEAR()`, etc.), the comparison uses computed values that the index was not built on. The engine must then evaluate every candidate row. Fixes: normalize at write time (store lowercase), use a case-insensitive collation, build a functional index, or rewrite as a direct range comparison (e.g. `created_at >= '2026-01-01' AND created_at < '2027-01-01'` instead of `YEAR(created_at) = 2026`). `LIKE '%foo'` (leading wildcard) defeats an index the same way, because it cannot anchor a binary search at the start.

Habit: run `EXPLAIN`/`EXPLAIN ANALYZE` before you guess whether a slow query uses the index. Look for `Seq Scan`/full scan vs `Index Scan`/`Index Only Scan`.

**Composite indexes and the leftmost-prefix rule.**
```sql
CREATE INDEX idx_orders_customer_status_created ON orders (customer_id, status, created_at);

-- USES the index — customer_id is the leading column
SELECT * FROM orders WHERE customer_id = 42 AND status = 'PENDING';

-- USES the index too — still a valid prefix
SELECT * FROM orders WHERE customer_id = 42;

-- DOES NOT use the index — status isn't the leftmost column, full scan.
SELECT * FROM orders WHERE status = 'PENDING';
```
A composite index is one B-tree. It sorts by the **first** column. Within each first-column value, it sorts by the second column. Within that, it sorts by the third. A query must filter on a left-to-right prefix: `(a)`, `(a, b)`, or `(a, b, c)`. It cannot filter on `(b)` alone. If a later predicate is wrapped in a function, that defeats the index for that column only. The engine can still use the index for the leading column(s), which narrows the scan to a subset. It then evaluates the remaining predicate row by row on that subset only, not on the whole table. Trade-off: each index speeds up matching reads but adds overhead to each `INSERT`/`UPDATE`/`DELETE` on the table.

**JOIN type decides whether unmatched rows survive. `= NULL` is never true.**
```sql
-- INNER JOIN: customers with zero orders vanish from the result entirely
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;

-- LEFT [OUTER] JOIN: every customer survives; o.* is NULL where there's no match
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
```
`INNER JOIN` keeps only rows that match on both sides. A customer with no orders has no row to match, so it silently disappears from the report. `LEFT JOIN` keeps every left-table row and fills unmatched right-side columns with `NULL`. `COUNT(o.id)` works correctly here because `COUNT` on a specific column skips `NULL`s, so those customers get `0`, not `1`.

A related NULL trap: `WHERE o.discount_code = NULL` **never matches any row, even rows where `discount_code` is `NULL`.** SQL `NULL` means "unknown", so `x = NULL` always evaluates to `UNKNOWN`, and a `WHERE` clause treats `UNKNOWN` as `false`. Use only `IS NULL` / `IS NOT NULL` for null checks. They test for the absence of a value directly.

**Foreign keys, referential integrity, and cascading deletes.**
```sql
CREATE TABLE customers (id BIGINT PRIMARY KEY, email VARCHAR(255) UNIQUE, ...);
CREATE TABLE orders (id BIGINT PRIMARY KEY, customer_id BIGINT REFERENCES customers(id), total DECIMAL, ...);

DELETE FROM customers WHERE id = 42;  -- fails: FK constraint violation
```
The FK's job is **referential integrity**: an `orders` row can never point at a `customer_id` that does not exist. Query convenience is not its job. For plain cleanup, delete child rows first, then the parent. That is the right pattern. It is the wrong choice when the child data is financial/legal/audit data that you must retain (e.g. a GDPR "delete my account" request on order history). The standard pattern there is the reverse: keep the orders and **anonymize the customer's PII**, or repoint `orders.customer_id` to a placeholder with `ON DELETE SET NULL`. Declarative options: `ON DELETE CASCADE` (auto-deletes children; convenient but dangerous), `ON DELETE SET NULL`, `ON DELETE RESTRICT` (blocks the delete; similar to the default).

**OFFSET pagination cost.**
```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 25 OFFSET 100000;   -- slow
SELECT * FROM orders ORDER BY created_at DESC LIMIT 25 OFFSET 0;        -- instant
```
Even with an index on `created_at`, the plan shows an **Index Scan**, not a full/sequential scan. The index does work. A B-tree can jump to a *key value*. It cannot jump to the *Nth row by position*. For `OFFSET 100000`, the engine walks the index entry by entry and discards the first 100,000 rows. The cost is `O(offset)` and is inherent to offset pagination. Fix: **keyset/cursor pagination**. Use `WHERE created_at < <last_seen_value> ORDER BY created_at DESC LIMIT 25`, and store the last row's value as the next page's cursor. If the cursor column is not unique, use a **compound cursor** `(created_at, id)` with a matching composite index. The tie-breaker makes each row's position unique.

**Soft delete vs. hard delete.**
```sql
-- Hard delete: unrecoverable
DELETE FROM orders WHERE id = ?;

-- Soft delete: reversible
UPDATE orders SET status = 'CANCELLED' WHERE id = ?;
```
Two trade-offs beyond "table growth" (a partial index on active statuses mostly removes that concern):
1. Each query on the table must filter out soft-deleted rows (`WHERE status != 'CANCELLED'`). If one report or join misses the filter, cancelled rows silently reappear as active.
2. Overwriting a shared lifecycle status column on "delete" can destroy the prior state needed for an accurate restore (an order might have been `PAID`, not a fixed reset value). Use a separate nullable `cancelled_at` timestamp next to the real status. Cancelling sets the timestamp and leaves status unchanged. Restoring clears the timestamp.

**Also worth reviewing:** normalization (1NF–3NF) vs. deliberate denormalization (state the accepted write cost). A full status-history table, when you need a durable audit trail of every past transition, not only the current state.

---
## 4. Spring Boot & JPA/Hibernate

**Constructor injection vs. field injection.**
```java
// Field injection — object exists in a half-built state, Spring reflects
// the dependency in afterward. Not final-able.
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
}

// Constructor injection — the field can be final: guaranteed non-null
// the instant the object exists.
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```
What changes:
- **`final` + fail-fast**: a missing bean fails application *startup* immediately with a clear error. Field injection can leave a `null` field that fails later as an NPE at the first call site.
- **Plain-JUnit testability**: `new OrderService(mockOrderRepository)` needs no Spring context. Field injection generally forces `@SpringBootTest` or reflection-based mocking.
- **SRP visibility**: a 7-parameter constructor is an obvious smell. Field injection hides the same smell.
- Bean lifecycle does not change. Spring manages both as singletons.

**`@Transactional` — rollback default.**
```java
@Transactional
public void transferFunds(Account from, Account to, BigDecimal amount) {
    debit(from, amount);
    log.info("debited");        // suppose this throws a checked IOException
    credit(to, amount);          // never reached — but the transaction still COMMITS,
                                  // because IOException isn't unchecked, so no auto-rollback
}

// Fix: opt in explicitly for checked exceptions.
@Transactional(rollbackFor = Exception.class)
```
Rule: `@Transactional` rolls back on **unchecked exceptions and `Error`s** by default. Checked exceptions need `rollbackFor`. It also protects atomicity only within **one method's** transaction. It does not coordinate two *separate* requests/transactions. That is a lost-update problem, solved by locking (see Concurrency).

**`@Transactional` self-invocation.**
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
        inventoryRepository.decrementStock(order);   // throws -> should roll back save(), but doesn't
    }
}
```
`@Transactional` does not work on the method itself. Spring wraps the **bean** in a dynamic proxy that intercepts calls to open/commit/roll back a transaction. The interception happens only when the call arrives **from outside the bean, through the proxy**. `this.saveOrder(order)` is a plain Java call on the raw object. It never touches the proxy, so no transaction opens. Fix: extract the method into a **separate injected bean**, so the call comes from outside the class and passes through the proxy.
```java
@Service
public class OrderPersistenceService {
    @Transactional
    public void saveOrder(Order order) { ... }
}
```
Reflex: when `@Transactional` "is not working", first ask *"Is this method self-invoked, called with `this.method()` or a bare unqualified call from another method in the same class?"*

**N+1 problem.**
```java
List<Order> orders = orderRepository.findAll();       // 1 query
for (Order o : orders) {
    o.getCustomer().getName();                          // N additional queries — one per order
}
```
Root cause: **lazy loading**. The `@ManyToOne`/`@OneToMany` association runs its query only when accessed, once per row. In logs: one query for the parent list, then N nearly identical queries for a related entity. Fixes: `JOIN FETCH` in JPQL, `@EntityGraph`, or batch fetching (`@BatchSize`). All reduce it to 1-2 queries.

**Entity gotcha — `equals`/`hashCode` on mutable, JPA-managed fields.**
```java
@Entity
@Data   // Lombok: generates equals()/hashCode() over ALL fields, including `id`
public class Customer {
    @Id @GeneratedValue private Long id;
    private String email;
    private String name;
}
```
Take a transient `Customer` (`id == null`) that you put in a `HashSet` or use as a `HashMap` key before you persist it. Its hashCode changes when Hibernate assigns a real `id` on save. This is the same mutable-hashCode-key problem as any mutable map key, now on an entity's transient-to-persistent transition. Fix: avoid blanket `@Data` on `@Entity`. Use `@Getter`/`@Setter` plus `@EqualsAndHashCode(of = "id")` (or hand-written equals/hashCode on `id` with a null-safe check). Secondary risk: an all-fields `toString()`/`equals()` can cause infinite recursion with a bidirectional relationship. Also, JPA requires a no-args constructor on `@Entity` classes for reflection-based instantiation. Lombok's `@Data` already includes an implicit `@RequiredArgsConstructor`. That is a no-args constructor *as long as there are no `final`/`@NonNull` fields*. It becomes a real gap only when you add such a field.

**Entity gotcha — missing `equals`/`hashCode` entirely. `id` is not a safe substitute before persist.**
```java
@Entity
public class Product {              // no equals()/hashCode() overridden at all
    @Id @GeneratedValue private Long id;
    private String sku;
    private String name;
}

List<Product> products = csvParser.parseAll(file);          // two rows, same SKU, none persisted yet
Set<Product> uniqueProducts = new HashSet<>(products);      // both still end up in the set — dedup fails
```
With no override, `equals()`/`hashCode()` fall back to `Object`'s identity-based defaults. Two separate `Product` instances are never "equal", whatever data they hold, so a `HashSet` cannot dedupe them by SKU. The tempting fix, to override on `id`, also fails here. Each object in this list is **transient** (`id == null`). All of them hash identically and compare equal through `null == null`. This silently collapses genuinely different products into one, instead of deduping only the true duplicates. The correct fix is a stable **business key** that is populated before persist (here, `sku`). The principle is the same as in the previous gotcha, from the opposite direction. That key *changes* after persist. This key *does not exist yet* before persist.

**`LazyInitializationException`.**
```java
@Service
public class OrderService {
    @Transactional
    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();   // Customer is lazy — not loaded yet
    }   // <-- transaction commits and Session closes HERE
}

@RestController
public class OrderController {
    public OrderResponse getOrder(@PathVariable Long id) {
        Order order = orderService.getOrder(id);
        return new OrderResponse(order.getCustomer().getName());  // throws LazyInitializationException
    }
}
```
The `Session` that a lazy proxy needs lives only as long as the `@Transactional` method's transaction. When that method returns, the transaction commits, the Session closes, and the returned entity is **detached**. An association not initialized *inside* the transaction can never be initialized afterward. Access to a non-id field on it throws. Access to only the FK scalar (e.g. `getCustomer().getId()`) is safe even when detached, because JPA can supply the id without a DB call. Fixes:
```java
// Fix 1 — eagerly load inside the still-open transaction
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.id = :id")
Order findByIdWithCustomer(@Param("id") Long id);

// Fix 2 — map to a DTO while the transaction/session is still open
@Transactional
public OrderDto getOrder(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    return new OrderDto(order.getId(), order.getCustomer().getName());
}
```
Know this anti-pattern by name: **"Open Session in View"**. It extends the Hibernate session across the whole HTTP request. It masks this exception at the cost of hidden N+1 queries at render time. Spring recommends `spring.jpa.open-in-view=false` and a fix of the real cause.

---

## 5. REST & API Design

**Idempotency. Ask: "If I send this exact request twice, is the end state different?"**

| Method | Idempotent? | Why |
|---|---|---|
| `GET` | Yes | Read-only, no state change |
| `PUT` | Yes (by contract) | Setting the same full state twice gives the same end state |
| `DELETE` | Yes (by contract) | Deleting an already-deleted resource is still "gone" |
| `POST` | No (by default) | It usually *creates* something. Sending it twice can create two things |

```java
// Cheap state-check guard for a PUT-like transition with a side effect
if (order.getStatus() == OrderStatus.CANCELLED) {
    return;  // already cancelled, no-op — makes a retried request safe
}
order.setStatus(OrderStatus.CANCELLED);
paymentGateway.refund(order);
```
Idempotency **keys** are the heavier tool. Use them when state alone cannot tell you "Did I already process this exact request?" The client generates a unique key per logical operation. The server stores "already processed this key" and returns the original result on retry, without re-running side effects. Stripe and other payment APIs use this pattern.

**Action endpoint vs. generic PUT.**
```
POST /api/orders/{id}/cancel     -- explicit: signals a real action with side effects
PUT  /api/orders/{id}            -- generic: hides that cancelling also refunds/releases inventory
```
A `PUT` can also be made idempotent (state check before re-running the side effect). The real decision driver is clarity of intent.

**Pagination.**
```
GET /api/v1/books                 -- fine at 500 rows, breaks at 2,000,000
GET /api/v1/books?page=0&size=25  -- bounded response size regardless of table growth
```
The fix belongs at both layers. The SQL query needs `LIMIT`/`OFFSET` or cursor pagination. The API contract needs `page`/`size` (or cursor) query params.

**DTOs vs. entities.** Returning an `@Entity` directly from a `@RestController` has three costs. It couples the DB schema to the public API contract (a schema change becomes a breaking API change). It can leak internal fields (password hashes, flags). It can trigger `LazyInitializationException` when it serializes an uninitialized lazy association outside a transaction. A separate DTO/response record decouples the two.

**Status-code precision in a multi-outcome scenario.**
```java
// success                  -> 200 OK
// order id not found       -> 404 Not Found
// order already shipped    -> 409 Conflict   <- NOT 400
```
`400` means the request itself is malformed. `409` means the request is well-formed but the *current state* of the resource conflicts with the action. It is the correct code for "already shipped, can't cancel." Pair this with a single consistent error envelope across every endpoint (normally a global `@ControllerAdvice`/`@ExceptionHandler` that maps specific exceptions to status + body). Add a stable, machine-readable error **code** (`"code": "ORDER_ALREADY_SHIPPED"`) next to the human message. A frontend can then switch on the code, not pattern-match message text.

**A `POST` that creates a resource. 200 vs. 201.**
```java
// Ambiguous: 200 only says "it worked," not "a new thing now exists"
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
    Order order = orderService.create(request);
    return ResponseEntity.ok(toResponse(order));            // 200 OK
}

// Correct: 201 signals creation, Location header points at the new resource
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
    Order order = orderService.create(request);
    URI location = URI.create("/orders/" + order.getId());
    return ResponseEntity.created(location).body(toResponse(order));   // 201 Created
}
```
`200` means only that the request succeeded. `201` means it succeeded *and created a new resource*. This is the more precise signal that a client wants from a creation endpoint.

**400 vs. 422 vs. 409. Three different reasons a request can fail.**
```
400 Bad Request          -- the request itself is malformed (missing field, wrong type, invalid JSON)
422 Unprocessable Entity  -- well-formed, but fails a business/semantic rule (an order with zero items)
409 Conflict              -- well-formed and valid, but conflicts with the resource's CURRENT STATE
```
Reflex question: *"Is something structurally wrong with the request (400)? Is it well-formed but breaking a rule (422)? Is the request fine, but the resource's current state does not allow it (409)?"* A common junior mistake is to use `409` for a plain validation failure like "order has no items". That is `400`/`422`, because no existing resource state conflicts. `409` is for cases like "already shipped, can't cancel."

**Resource modeling. Path vs. query parameters.**
```
GET /customers/42/orders             -- one customer's orders: id in the PATH (scopes the collection)
GET /customers/42/orders?since=2026-07-28  -- query param filters WITHIN that scope

GET /orders?since=2026-07-28         -- "all customers": no single scope, route reverts to flat + query filter
```
General rule: **path parameters identify which specific resource/sub-collection you are scoped to. Query parameters filter or modify the result set within that scope.** Prefer self-documenting query param names (`?since=2026-07-28` or `?days=7`) over custom duration syntax (`?range=7D`).

**Also worth reviewing:** API versioning for a field change that does not break existing clients. Jackson basics (`@JsonIgnoreProperties(ignoreUnknown = true)`, date format configuration).

---

## 6. Concurrency & Locking

**In-process race condition (lost update) on a shared counter.**
```
Shared field: private int count = 0;   // count++ is NOT atomic — it's 3 steps:
                                        // 1. read count  2. add 1  3. write count back

Thread A: read count (0)
Thread B: read count (0)          <- both threads see the same starting value
Thread A: write count = 1
Thread B: write count = 1          <- B's write clobbers A's — one increment is lost
```
Fixes, in order of preference for a simple counter:
```java
// Best for a single variable — lock-free compare-and-swap
private final AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();

// Works, but blocks other threads — needed once the invariant spans MORE than one field
private int count = 0;
public synchronized void increment() { count++; }
```
Name the concept explicitly: this is a **race condition**, specifically a **lost update**.

**Thread-unsafe collections under concurrent access.**
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
A plain `HashMap` gives **no thread-safety guarantee**. `put()` can trigger an internal resize/rehash of the bucket array. Two threads that call `put()` at the same instant race on that array. They can silently drop or overwrite entries, or return `null` for a key that should exist. This is also what throws `ConcurrentModificationException` in the cases that are caught: a fail-fast check on an internal modification counter. This is a structural-corruption bug, not a cache-staleness problem, so cache invalidation does not fix it. Idiomatic fix: `ConcurrentHashMap`, built for concurrent access:
```java
private final Map<Long, Product> cache = new ConcurrentHashMap<>();

public Product getProduct(Long id) {
    return cache.computeIfAbsent(id, key -> productRepository.findById(key).orElseThrow());
}
```
`computeIfAbsent` also fixes the check-then-act race (`containsKey` → `get` → `put`). It makes the lookup-or-compute atomic per key. Reflex: a mutable `Map`/`List`/`Set` field in a Spring singleton bean is touched by many request threads. Use the matching `java.util.concurrent` type before you hand-write a `synchronized` block.

**Tell the concurrency problem shapes apart. This is the highest-leverage distinction to learn.**

| Shape | Symptom | Correct fix | Wrong fix |
|---|---|---|---|
| **In-process race**: one shared mutable field, many threads, same JVM | Lost updates on a counter under concurrent load | `AtomicInteger` / `synchronized` | — |
| **Lost update across two transactions**: two separate HTTP requests each read-then-write the same DB row | One request's edit is silently overwritten by another's later save | Optimistic locking (`@Version`) or pessimistic locking (`SELECT ... FOR UPDATE`) | `@Transactional` alone. It wraps one method and does not coordinate two separate transactions |
| **Duplicate-insert race**: two requests both pass a "does this exist?" check before either inserts | Two rows with the same email after concurrent signups | DB-level `UNIQUE` constraint (no lock needed, nothing to lock yet) | Pessimistic locking. No existing row exists for `SELECT ... FOR UPDATE` |
| **Deadlock via lock-order inversion**: two transactions each pessimistically lock the same two or more rows, in opposite orders | Both transactions fail with a real DB deadlock exception, almost instantly | Enforce one consistent lock-acquisition order in every code path that locks those rows | Explaining it as ordinary lock contention/timeout, or switching wholesale to optimistic locking |

Always ask first: **"Is this racing inside one process (needs `synchronized`/atomics), or racing across time through the database (needs a schema constraint or transactional locking)?"**

**Optimistic locking.**
```java
@Entity
class Customer {
    @Id Long id;
    String phone;
    @Version
    Long version;   // Hibernate adds "AND version = ?" to every UPDATE, bumps it on every write
}
// If someone else already bumped the version, 0 rows match -> OptimisticLockException.
// Catch it: retry the operation or report the conflict to the user.
```
Fits read-heavy, low-conflict situations. It does not block and is cheap, but you must handle the conflict exception.

**Pessimistic locking.**
```sql
BEGIN;
SELECT * FROM seats WHERE id = ? FOR UPDATE;   -- row-level lock; other transactions
                                                 -- locking this row must WAIT
-- ... book the seat ...
COMMIT;                                          -- lock released
```
Fits short, high-conflict operations (e.g. last-seat booking) where losing the conflict is expensive. It trades throughput for safety. It can cause timeouts under heavy load. It adds deadlock risk if two transactions lock many rows in different orders. Keep the transaction short.

**Optimistic vs. pessimistic under extreme contention. The wasted-work argument.**
```
Single unit left, 500 concurrent buy attempts:

Optimistic: all 500 read the row, race to UPDATE ... WHERE version = ?,
            1 wins, 499 lose -> each loser retries the WHOLE read-modify-write
            cycle, discovers the item is STILL gone, retries again... wasted DB round trips.

Pessimistic: all 500 queue on SELECT ... FOR UPDATE for that ONE row,
             each runs in turn, checks stock, decrements or reports "sold out" -
             no retry loop, just a wait.
```
The "short, high-conflict → pessimistic" rule above is an argument about *wasted work*. Under heavy contention over one scarce resource, most optimistic-locking retries fail before they run. Paying for a queue upfront is cheaper than paying for repeated failed attempts. Be precise about scope: `SELECT ... FOR UPDATE` locks the **row**, not the table or the database. "Locking the DB" is a common but wrong simplification.

**Deadlock via lock-order inversion.**
```java
// Endpoint A: locks orders first, then inventory
// Endpoint B: locks inventory first, then orders
```
```
Txn A: locks orders(7)
Txn B: locks inventory(3)
Txn A: tries to lock inventory(3) -> BLOCKS, waiting on Txn B
Txn B: tries to lock orders(7)    -> BLOCKS, waiting on Txn A     -- circular wait
```
A **deadlock** is a circular wait. Each transaction holds what the other needs next, so the cycle has no way to progress. It is *not* ordinary lock contention. Contention means many transactions wait in one direction in a queue. It resolves eventually and is never flagged as a deadlock. A database's deadlock detector finds a real deadlock almost immediately (milliseconds, not a load-dependent timeout). It kills one transaction to break the cycle. Fix: enforce **one consistent lock-acquisition order** in every code path that touches the same rows (always lock `orders` before `inventory`, or lock by ascending row ID). Do not switch wholesale to optimistic locking.

**Distributed locking.** A `synchronized` lock is tied to an object inside **one JVM process**. Two app instances are two separate JVMs. Each has its own independent copy of the lock object. Neither sees the other's lock, so two threads on different instances can enter a "synchronized" method at the same instant. In-process tools (`synchronized`, `AtomicInteger`) coordinate only threads *within a single process*. With more than one instance, they silently stop working. The Redis `SET key value NX PX <ttl>` pattern (set-if-not-exists with an expiry) is the basic idea for a cross-instance lock. Redis becomes the one coordinator that every instance defers to. Often you need no lock at all. A unique constraint, or an atomic `UPDATE ... WHERE version = ?`, already gives the guarantee for "the same request might get delivered twice."

---
## 7. Testing & Debugging

**Debug an intermittent, timing-dependent bug. The sequence matters:**
1. **Reproduce reliably first.** A debugger is close to useless until you can trigger the bug on demand. If the report includes a trigger ("double-click," "fast," "sometimes"), recreate it mechanically (script two near-simultaneous requests). Do not click quickly by hand.
2. **Choose tools deliberately for timing bugs.** A debugger breakpoint changes the timing of everything around it. That can make a race condition disappear (a "Heisenbug"). Logging observes without altering timing, and can run in production where the bug actually happens.
3. **Use structured logging, not `println` + redeploy.** A real logger has levels (`DEBUG`/`TRACE`) and structured context: request ID, timestamp, user/order ID, thread/instance ID. You can then isolate a specific failing request.
4. **Form an explicit hypothesis for *why* the bug is intermittent** before you pick a tool. Candidate hypotheses for a "sometimes" bug: a non-idempotent endpoint hit twice (double-submit), an exception silently swallowed inside a loop, a job running on two instances/threads at the same time.
5. **Binary-search debugging** once you can reproduce. Narrow down *where* in the flow the bug starts. Check state at successively narrower points.

**Not every intermittent bug is timing-based. Resource/threshold effects need a different hypothesis.**
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
A deep category tree intermittently throws `StackOverflowError` with **identical input** each time. This is not a logic bug or a timing race. It is a **resource/threshold bug**. The recursion depth sits at the edge of the JVM's available stack space. Small, incidental variations in how much stack the surrounding call chain already used (which pooled thread handled the request, how many filter/proxy layers ran first) tip it over the edge on some runs and not others. Confirm by logging the recursion depth reached and comparing it to the JVM's default stack size. Do not assume "the tree is too deep" without asking why it is *sometimes* too deep. Fix: convert the recursion to an **iterative** traversal with an explicit stack (`Deque<Category>`). This removes the dependency on JVM stack size. Raising `-Xss` only raises the threshold and does not remove the fragility.

**Mocking anti-pattern: over-mocking / "testing the mock, not the code."**
```java
@Test
void appliesDiscount() {
    ProductRepository mockRepo = mock(ProductRepository.class);
    PricingEngine mockPricing = mock(PricingEngine.class);
    DiscountCalculator mockCalculator = mock(DiscountCalculator.class);
    when(mockRepo.findById(1L)).thenReturn(product);
    when(mockPricing.getBasePrice(product)).thenReturn(100.0);
    when(mockCalculator.calculate(100.0, 0.1)).thenReturn(90.0);   // stubbed

    DiscountService service = new DiscountService(mockRepo, mockPricing, mockCalculator);
    double result = service.applyDiscount(1L, 0.1);
    assertEquals(90.0, result);   // checks the exact value that was stubbed in
}
```
The test mocks every collaborator, and each mock returns exactly the value that the assertion checks. The tell: delete `DiscountService`'s real implementation and replace it with `return 90.0;`. The test still passes. A test that survives deleting the logic it should verify does not test that logic. Fixes: add `verify(mockPricing).getBasePrice(product)` to confirm orchestration (the service calls collaborators with the right arguments). Or recognize that mocking every dependency is itself the smell, and test the real math against a real or minimally faked collaborator. Reflex: every dependency is mocked *and* the stub's return value matches the assertion. Ask: "What real behavior can still fail if I break something?"

**Test-driving a bug fix.** Before you fix a reported bug, do not only "check/run the existing tests". A bug that reached production almost certainly has **no existing test** for that exact case. Otherwise CI would have caught it. Correct sequence:
```java
// 1. Write a NEW test that encodes the exact reported bug
@Test
void cancelOrder_whenAlreadyShipped_doesNotTriggerRefund() {
    Order order = anOrderWithStatus(OrderStatus.SHIPPED);
    orderService.cancel(order.getId());
    verify(paymentGateway, never()).refund(any());
}
// 2. Run it against the CURRENT (buggy) code — it should FAIL (proves the bug is genuinely reproduced).
// 3. Make the actual fix.
// 4. Re-run — it should now PASS (proves the fix works).
// 5. Keep the test in the suite as a permanent regression guard.
```

**Flaky tests, weak assertions, one behavior per test.**
```java
@Test
void testDiscount() {
    Order order = new Order();
    order.addItem(new Item("Widget", 100.0));
    order.addItem(new Item("Gadget", 50.0));
    order.applyCoupon("SAVE10");
    assertTrue(order.getTotal() < 200.0);        // passes even with ZERO discount applied
    assertNotNull(order.getAppliedCoupon());
    assertEquals(2, order.getItems().size());    // fails intermittently, no local repro
}
```
Three distinct problems:
1. **One test, three behaviors.** A vague method name and many unrelated assertions mean you must read the stack trace closely to see what broke. Fix: one behavior per test, named for that behavior.
2. **A loose assertion does not test what it claims.** `assertTrue(total < 200.0)` passes even if the coupon does nothing. Fix: assert the exact expected value with a small tolerance, `assertEquals(135.0, order.getTotal(), 0.001)`.
3. **A flaky assertion with zero code changes and no local repro is a test-isolation smell, not a coincidence.** Likely cause: shared mutable state leaks across tests (a static/shared fixture reused without reset, a test DB not rolled back, parallel execution that touches shared state). Fix: find and remove the shared state (fresh data per test, `@BeforeEach` reset). Never loosen the assertion. Never re-run until green.

Reflex for "test fails sometimes, no code changes, can't repro locally": ask *"Is this test's setup actually isolated?"* before you assume a timing race. Plain shared-fixture bugs are usually isolation problems, not concurrency problems.

**Unit vs. integration tests. The real distinguishing question.**
```java
// Slow "unit" test that actually boots the whole app and hits a real DB
@SpringBootTest
class PriceCalculatorTest {
    @Autowired PriceCalculator calculator;
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
        verify(priceRepository).findBasePrice(any());   // verify orchestration too
    }
}
```
The distinguishing lens is not "how much business logic" a method has (almost every method has some). It is **which boundary the test verifies**. Pure logic in isolation (mock every collaborator) is a unit test. A real interaction across a boundary (does this query return the right rows against a real schema) is an integration test. The **testing pyramid**: many fast, isolated unit tests at the base. Fewer integration tests that validate real boundaries. Very few end-to-end tests at the top. For each test, ask: "Which collaborators do I trust, and which one do I verify?" Mock everything you trust. Run only the thing under test for real.

---

## 8. Web Security & Frontend Basics

**CORS. Full mechanism.**
```
localhost:3000 (frontend)  ---fetch()--->  localhost:8080 (backend)
```
A different **port** already means a different **origin** (scheme + host + port must all match). This triggers the browser's **same-origin policy**. Critical nuance: **the request still reaches the backend and the backend processes it**. The browser blocks frontend JavaScript from *reading the response*, unless the server's headers explicitly allow it.
```java
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
// Or per-controller: @CrossOrigin(origins = "http://localhost:3000")
```
**Preflight `OPTIONS` request**: for "non-simple" requests (JSON body, custom headers, methods beyond simple GET/POST-form), the browser first sends an automatic `OPTIONS` request to ask permission. If the backend does not answer correctly, the browser never sends the real request. This is why a blocked call can show **zero log lines** in the controller. Production gotcha: browsers reject `allowedOrigins("*")` combined with `allowCredentials(true)` outright. Use an explicit domain whitelist in production.

**Request lifecycle. Type a URL and press Enter:**
1. **DNS resolution** — domain name → IP (browser cache → OS cache → resolver → root/TLD/authoritative servers).
2. **TCP handshake** — SYN → SYN-ACK → ACK. It establishes a reliable connection before any data is exchanged.
3. **TLS handshake** (if HTTPS) — certificate exchange/verification and encryption-key negotiation, before any HTTP request is sent.
4. **HTTP request/response** — the browser sends the request line + headers, the server responds, the browser parses and renders.

`ERR_CONNECTION_REFUSED` happens at step 2. The TCP handshake was actively rejected (nothing listening, service down, firewall). This is fundamentally different from a *timeout*, where the packet went unanswered and was not refused. DevTools → Network tab → "Timing" breakdown shows the real phases with real numbers.

**React data fetching pattern.**
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

**SQL injection.**
```java
// Vulnerable: string concatenation lets user input become part of the SQL text itself
String query = "SELECT * FROM products WHERE name = '" + request.getParameter("name") + "'";

// Fixed: PreparedStatement — structure and data are sent to the DB separately
String query = "SELECT * FROM products WHERE name = ?";
PreparedStatement stmt = connection.prepareStatement(query);
stmt.setString(1, request.getParameter("name"));
```
A malicious `name` value can break out of the string literal and inject more SQL (e.g. a `JOIN`-based attack that exfiltrates data from unrelated tables). Why `PreparedStatement` fixes it: the database compiles the query structure *before* it binds the value. The DB never re-parses user input as SQL syntax, whatever characters (`'`, `;`, `--`) it contains. Server-side input validation is fine as defense-in-depth but is **not** a substitute. Attackers routinely bypass blacklist-style validation.

**Cookie tampering vs. JWT signing.**
```
// Insecure: user=John;role=admin  stored directly in a plain cookie
```
The sharpest risk of a plain cookie that stores role/identity is not mainly an outside attacker who reads it (confidentiality). The *same logged-in user* can open DevTools and edit `role=admin` onto their own cookie to escalate privileges. This is an **integrity/tampering** problem, and it needs no outside attacker. A JWT fixes this because it is **cryptographically signed** (HMAC/RSA). If a client edits any claim without the server's secret key, the signature no longer matches on verification, and the server rejects the request. "Attach it to a header and validate server-side" alone is not the mechanism. The signature check is. The sibling standard answer is a traditional **server-side session**. The cookie holds only an opaque random session ID. The actual role/user data lives server-side (DB/Redis) and is looked up per request. Trade-off: JWT avoids a per-request lookup but cannot be revoked instantly. A session ID is trivially revocable but costs a lookup.

**Session cookies vs. JWTs. "Stateful vs. stateless" is the first-level answer, but it does not decide the choice.**
```
                        Session cookie                       JWT
Attached how?           browser auto-sends via Cookie header  client code manually sets Authorization header
Where's the state?      server-side (DB/Redis) — cookie is    entirely inside the token itself —
                         just an opaque lookup key             server tracks nothing
Revoke instantly?        yes — delete the session record       no — valid until expiry no matter what;
                                                                needs a blocklist or short expiry + refresh token
Theft surface           can be `httpOnly` — invisible to       usually stored/read by JS (e.g. `localStorage`)
(if stolen via XSS)     an XSS-injected script                 to attach to headers — directly readable by it
```
"Stateful vs. stateless" is a correct label. Two other properties drive the choice in practice. **Revocability**: can you kill it server-side right now, e.g. for "log out everywhere" or a compromised account? **Theft-resistance**: where does the token sit, and can injected JS read it? Neither scheme is inherently "better". A backend can run both at once (session for the first-party web app, JWT for mobile/other services) through separate auth filters. That is routine, not a conflict.

**Also worth reviewing:** never trust client-side-only validation (hiding a button is not security). Always re-validate server-side.

---
## 9. System Design & Architecture Basics

**Layered architecture.** Controller (API surface) → Service (business logic) → Repository (persistence). Each layer has clear ownership boundaries.

**Caching. Cache-aside pattern.**
```
1st request:  client -> app -> (cache MISS) -> DB -> app writes to cache -> client
2nd request:  client -> app -> (cache HIT)  -> client   (DB never touched)
```
A too-long TTL risks overselling on fast-changing data. Cached "5 in stock" goes stale while real stock hits 0. A `quantity < 0` guard is a necessary safety net. Vocabulary:
- **Cache stampede / thundering herd** — the too-short-TTL failure mode. Many requests miss the cache at the same instant (a hot key expires) and all hit the DB at once.
- **Invalidate-on-write** — the complement to TTL. When stock/price changes, evict/update the cache entry immediately. Do not wait for TTL expiry.
- Rule of thumb: long TTL for slow-changing fields (description, images). Short TTL or invalidate-on-write for fast-changing/high-stakes fields (stock, price).

**Bottleneck reasoning.** The reliable heuristic: **"Which operation happens far more often, reads or writes?"** In a URL shortener, redirect/lookup traffic far exceeds create traffic (often 100:1 or more). The bottleneck is the read/redirect path. Fix it with a cache (e.g. Redis) in front of the DB for hot short codes. Add a unique index on `short_code` so the lookup is not a table scan. Redirect mechanics: HTTP `301` (permanent) or `302` (temporary) with a `Location` header.

**CRUD schema design checklist.** Before you finalize the schema, walk every planned endpoint against it: **list, get-one, create, update, delete**. Does the schema have what each of the five needs to read or write? Common misses: no `completed` boolean for a "mark complete" API. No list/search endpoint at all.

**Sync (direct call) vs. async (message queue).**
```
Direct call (sync OR async style)          Message queue
caller ---request---> callee               caller --publish("OrderPlaced")--> queue ---> consumer ---> email service
caller still needs callee's result          caller returns immediately, never waits on
or has to handle its failure right now      the consumer or downstream service at all
```
`async/await` or a non-blocking `HttpClient` call is still a **direct call**. It does not block the calling thread, but the caller stays coupled to that service's availability. A **message queue** (RabbitMQ/Kafka/SQS) is a stronger, different kind of decoupling. The caller publishes and returns immediately. A separate consumer processes the message when it can, even if the downstream service is down. Retries on transient failures come for free. Decision rule: **"Does the caller need the result before it can respond to the user?"** Payment: yes, direct call. Confirmation email: no, queue candidate.

**Stateless services.**
```
1 instance:              login -> writes session to in-process Map -> works fine
2 instances, round-robin: login -> server A's Map -> next request routed to server B -> B's Map has nothing -> "not logged in"
```
General principle: services should be **stateless**. Anything in a singleton/in-process field (a session map, an in-memory cache, a rate-limiter counter) exists only on the instance that wrote it. It silently breaks when more than one instance exists. Standard fix: move the store to Redis (or a shared DB), so every instance reads and writes the same source of truth. Alternative: **sticky sessions** (the load balancer routes the same client to the same instance by cookie/IP hash). They are simpler to add but fragile. A restarted instance loses every session pinned to it. They can also unbalance load.

---

## 10. Tooling: Git, Build & Environment

**Commit hygiene.**
```
fix: reject empty bearer token in auth middleware
chore: rename OrderProcessor -> OrderPipeline
refactor: reformat build.gradle
```
Split a mixed commit by logical concern. Use conventional-commit prefixes. Why it matters beyond style:
- **`git blame`/`git bisect` clarity** — a future regression search lands on a commit whose message describes what changed there. A "fixes stuff" commit that bundles two unrelated changes does not help.
- **Revert safety** — if one change is wrong and needs a revert, a bundled commit forces you to revert unrelated changes too.
- **Review signal vs. noise** — a reviewer can skim past a security-sensitive change that sits next to cosmetic renames.

Reflex: *"If I had to revert or `git blame` just this one change in three months, could I isolate it cleanly?"* If not, split the commit. `git add -p` splits hunks within a single file.

**Secrets in git.** A committed secret **stays in git history forever**, even after a later commit deletes the file. Anyone with clone access can find it in an old commit. Rotation is therefore mandatory once a secret has been pushed even once. Add `.env` to `.gitignore` immediately. Commit a `.env.example` with placeholder keys/values so onboarding still works. Real secrets belong in a runtime-injected source: the CI/CD secret store or a secret manager (Vault, AWS Secrets Manager). Never hardcode them in the repo, even in a "local-only" file that you might commit by accident. Bonus habit: secret scanning in CI (`gitleaks`, GitHub secret scanning) catches this before merge.

**Maven/Gradle dependency conflicts.**
```bash
mvn dependency:tree            # prints the resolved graph, flags dropped versions as "(conflict)"
./gradlew dependencies         # Gradle equivalent
./gradlew dependencyInsight --dependency jackson-databind
```
Maven's resolution rule is **nearest-wins**. The declaration closest to your own project in the tree wins, whichever version is newer. A transitive dependency three levels deep can silently lose to an older version declared one level up. Fixes: pin the version explicitly in your own `pom.xml` (a direct declaration always wins over a transitive one). Use `<dependencyManagement>` to centralize versions across a multi-module project. Or `<exclude>` the unwanted transitive version.

**Resolve a merge conflict. Combine both sides. Do not pick one.**
```
<<<<<<< HEAD
// your current branch's version
=======
// incoming version from main
>>>>>>> main
```
Suppose a teammate restructured a method and you separately added a new field/behavior inside it. The correct resolution usually has **both** changes, not one or the other. Picking a side wholesale is how real work quietly disappears. Process: read both blocks, combine them manually into one correct version, delete the markers, and **test/run it** (a syntactically clean merge can still be semantically wrong). Then `git add`/`git commit` (merge) or `git rebase --continue` (rebase). An IDE's 3-way merge view makes "combine both" more obvious than raw markers. `git log -p <file>` or `git blame` on the conflicting lines often shows what each side changed, before you need to ask a teammate.

**Environment config across dev/staging/prod.** A `Could not resolve placeholder 'DB_PASSWORD'` failure in staging (but not locally) usually means something local-only supplied the value on the laptop (a `.env` file, an IDE run config, an exported shell variable). Nobody set it up in staging. Implicit local machine state did not travel with the app. Two-lens fix:
- **Config that differs by environment but is not secret** → Spring profiles: `application-dev.properties` / `application-staging.properties` / `application-prod.properties`, switched with `spring.profiles.active=<profile>`.
- **Actual secrets** → platform-injected runtime environment variables (CI/CD secret store, Kubernetes Secret, AWS Secrets Manager). Never use a repo file, even a per-environment one. `.env` is fine for local dev convenience, but add it to `.gitignore` the moment it holds a real secret. Commit a `.env.example` with placeholders instead.

**Merge vs. rebase.**
```
git merge main            git rebase main
     A---B---C (main)          A---B---C (main)
    /         \                         \
D---E (yours)  M  <- new merge commit    D'---E' (yours, replayed
                   with 2 parents           on top of C, NEW hashes,
                   (history stays          straight-line/linear
                   non-linear)             history)
```
`merge` leaves existing commits untouched and adds a new two-parent merge commit. History is non-linear and nothing is rewritten. `rebase` replays your commits on top of `main`'s current tip one at a time. The content is the same, but every commit gets a **brand-new hash**, and the history is linear. Merge and rebase both preserve each commit's original **author metadata** identically. `git blame`/`git log --author` works the same either way. Rebase changes only the commit hash, not authorship. The real reasons to prefer rebase: a cleaner linear history and an easier-to-read PR diff.

**Rule to memorize: never rebase (and force-push) a branch that other people have already pulled and built on.** Rebase creates new hashes for every commit. A teammate who pulled your branch and committed on top of the *old* hashes now diverges at the hash level. A force-push pulls the rug from under their local branch (their commits' parents no longer exist upstream). This risks lost or duplicated work. It is a history-rewrite problem, not an ordinary merge conflict. Rebase freely on a private branch that only you use. Once the branch is shared, prefer merge, or coordinate the rebase explicitly with everyone who has it.

**Respond to PR review feedback you disagree with. Evidence over assertion.**
1. Check the code, not your memory. Trace whether the flagged concern (e.g. a second code path) actually calls the same logic you changed, or a separate implementation.
2. Turn the check into evidence, not only a claim. Point to the specific file/line so the reviewer can verify it.
3. Better: write a quick test that exercises the questioned path with the input that triggered the original issue. It gives concrete proof and a permanent regression guard either way.
4. If you are wrong, this same process shows it *before* you argue in the thread, not after.

Reflex: resolve a code-review disagreement with evidence (code, tests, logs). Never restate your own confidence level.