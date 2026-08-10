# Backend Interview Knowledge Reference

A pure study reference distilled from mock interview sessions — concepts only, organized by topic, with code examples where they clarify the mechanism.

---

## 1. Core Java & CS Fundamentals

**Checked vs unchecked exceptions — the hierarchy decides it, not intent.**
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
Reflex: any `Reader`/`Stream`/`Connection`/`Socket` opened with `new` inside a method → ask "does this need try-with-resources?" (It needs to implement `AutoCloseable`, which nearly all I/O classes do.)

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
Decision rule: use `Set<K>` when you only need "does this exist?"; use `Map<K, V>` when you need the actual object back.

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
Rule of thumb: whenever `LocalDateTime.now()` touches anything cross-timezone, ask "whose 9am is this?" If the answer depends on who's reading it, you need a `ZoneId` in the picture.

**`hashCode`/`equals` contract — why overriding one without the other silently breaks `HashSet`/`HashMap`.**
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
`HashSet.add()` does **not** compare the incoming object against every existing element with `.equals()`. It first calls `hashCode()` to compute a **bucket index** (`hash % numBuckets`), and only compares `.equals()` against whatever is already sitting in that *same* bucket. Two `Customer` instances with the same `customerId` but loaded as separate objects will, with the default identity-based `hashCode()`, land in different buckets — `.equals()` never even gets called. The contract: *"if two objects are equal via `.equals()`, they must return the same `hashCode()`."* Reflex: any time you override `equals()`, generate `hashCode()` in the same action.

**Mutating a `HashMap` key after insertion — bucket location is fixed at `put()` time, never recomputed.**
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
`HashMap` computes `hashCode()` once, at `put()` time, to pick a bucket — the entry sits there permanently. `get()` recomputes `hashCode()` fresh and only searches the bucket the *current* hash points to. `String` never hits this because it's immutable (and caches its computed hash after the first call). Reflex: any class used as a `Map`/`Set` key needs every `equals()`/`hashCode()`-relevant field to be effectively immutable for the object's lifetime as a key.

**Integer caching / autoboxing — `==` "works" by accident for small values.**
```java
Integer a = 100;
Integer b = 100;
System.out.println(a == b);   // true

Integer c = 200;
Integer d = 200;
System.out.println(c == d);   // false
```
`Integer a = 100` autoboxes through `Integer.valueOf(int)`, not `new Integer(100)`. `valueOf()` maintains an internal cache (`IntegerCache`) for values **-128 to 127** — any value in that range returns the *same cached object* every time, so `==` (reference comparison) happens to return `true`. Outside that range, `valueOf()` allocates a new object each call, so `==` correctly says "different objects" — which isn't what the code meant to ask. Fix: **always use `.equals()`** to compare wrapper types by value. Reflex: any `==` next to a wrapper type (`Integer`, `Long`, `Boolean`, `Character`) is suspect.

**Also worth knowing:**
- `HashMap` internals: collision handling (linked list, or red-black tree since Java 8 once a bucket gets large enough), resizing/rehashing on load factor.
- Binary search preconditions: the collection must already be sorted.

---

## 2. OOP, Design Patterns & Low-Level Design

**YAGNI judgment.** The question to ask before reaching for a pattern: *"What specifically is going to vary, and how often?"* If you can't name a concrete future change, a pattern is premature. Strategy/Factory over 2-3 static, unchanging branches is usually overkill.

**Pattern definitions — identify a pattern by the problem it solves, not its shape:**

| Pattern | Solves | NOT this |
|---|---|---|
| **Strategy** | Swap an *algorithm/behavior* at runtime via a common interface | Just "interface + multiple implementers with a shared field" — that's plain polymorphism |
| **Facade** | Simple unified entry point to a *complex, multi-class subsystem* (e.g. `CheckoutFacade` coordinating `InventoryService`, `PaymentGateway`, `NotificationService`) | "Putting everything into 1 class" — that's a **God class** anti-pattern |
| **Factory** | Centralize *object creation* so callers don't need to know which concrete class to instantiate | Just "a class with a `create()` method" without a real decision inside it |
| **Singleton** | Guarantee exactly one shared instance (Spring beans are singleton-scoped by default, managed by the container) | Manually writing private constructor + static `getInstance()` — in Spring, let the container do this |
| **Observer** | Decouple "something happened" from "who reacts to it" (`ApplicationEvent`/`@EventListener` in Spring) | Direct method calls between unrelated classes |
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
Naming rule: name a strategy interface for the **verb** it performs (`PayCalculator`, `NotificationSender`) — not a noun.

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
class ContractorPayCalculator implements PayCalculator { ... }

class Employee {
    private final PayCalculator payCalculator;
    double calculatePay() { return payCalculator.calculate(this); }
}
```
The OOP pillar doing the real work is **polymorphism** — the concrete `PayCalculator` implementation runs at runtime through the interface, with no if/else on employee type. `Employee` can still stay a base class for shared fields (name, id) — abstract-class-for-shared-state and interface-for-varying-behavior aren't mutually exclusive.

**Dependency Inversion Principle.** High-level modules shouldn't depend on low-level modules directly — both should depend on abstractions.
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
The payoff isn't just "less code to change if the constructor changes" — it's that you can now swap implementations (add PayPal later) and unit-test business logic without ever hitting a real payment API. In Spring: mark the concrete class `@Service` implementing the interface, inject the interface type via constructor, and the container wires the concrete bean automatically.

**LLD process — separate the two questions.** For any low-level design exercise (parking lot, vending machine, library system), write two separate lists before sketching classes: *nouns with state* (what data exists, and does it persist across requests or reset per-transaction) and *nouns with behavior*. Before reaching for an interface, name the concrete thing that would actually differ between implementations — if nothing behavioral varies (e.g. different drink types that only differ in name/price), that's one concrete class/record with multiple instances, not one interface per variant. Don't reuse "Factory" for what's actually inventory/repository responsibility (tracking and mutating stock state is not object creation).

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
Why it matters beyond style: the invariant ("can only ship if paid and has a positive total") now lives in exactly one place, instead of depending on every caller remembering to check it correctly. Reflex: any time a service method computes something purely from one object's own getters with no other collaborator involved, ask "should that object just do this to itself?"

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
`PRIMARY KEY` only protects the key column. An app-level "check if it exists, then insert" is not equivalent to a real guarantee, because it isn't atomic.

**The classic duplicate-insert race, and why only a DB constraint closes it:**
```
Time  Request A                       Request B
t0    SELECT ... WHERE email=X        
t1                                    SELECT ... WHERE email=X
t2    (no rows found)
t3                                    (no rows found)
t4    INSERT (email=X)
t5                                    INSERT (email=X)   -- duplicate!
```
Both requests read "not found" before either writes. Only a **DB-level `UNIQUE` constraint** closes this gap — enforced atomically at insert time. The second `INSERT` gets rejected, and the app catches the constraint-violation exception. There's no row to lock yet at this point, so pessimistic locking doesn't apply — that tool protects *existing* rows, not the absence of one.

**Index mechanics and how to defeat one.**
```sql
-- Index on email is USED
SELECT * FROM users WHERE email = 'someone@example.com';

-- Index on email is DEFEATED — the DB must compute UPPER(email) for every
-- row before comparing, so it can't use the b-tree ordering built on raw values.
SELECT * FROM users WHERE UPPER(email) = UPPER('someone@example.com');
```
A B-tree index keeps *raw* column values in sorted order so lookups/range scans are `O(log n)`. Wrapping the column in any function (`UPPER()`, `YEAR()`, etc.) moves the comparison to computed values the index was never built on, forcing the engine to evaluate every candidate row. Fixes: normalize at write time (store lowercase), use a case-insensitive collation, build a functional index, or rewrite as a direct range comparison (e.g. `created_at >= '2026-01-01' AND created_at < '2027-01-01'` instead of `YEAR(created_at) = 2026`). `LIKE '%foo'` (leading wildcard) defeats an index the same way — it can't anchor a binary search at the start.

Habit: run `EXPLAIN`/`EXPLAIN ANALYZE` before guessing whether a slow query is hitting the index — look for `Seq Scan`/full scan vs `Index Scan`/`Index Only Scan`.

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
A composite index is a single B-tree sorted by the **first** column, then the second *within* each value of the first, then the third *within* that. A query must filter on a left-to-right prefix — `(a)`, `(a, b)`, or `(a, b, c)` — never `(b)` alone. If a later predicate is wrapped in a function (defeating index usage for that column specifically), the engine can still use the index for the leading column(s) — the scan narrows to that subset, then evaluates the remaining predicate row by row on just that subset, not the entire table. Trade-off: every index speeds up matching reads but adds overhead to every `INSERT`/`UPDATE`/`DELETE` on that table.

**Foreign keys, referential integrity, and cascading deletes.**
```sql
CREATE TABLE customers (id BIGINT PRIMARY KEY, email VARCHAR(255) UNIQUE, ...);
CREATE TABLE orders (id BIGINT PRIMARY KEY, customer_id BIGINT REFERENCES customers(id), total DECIMAL, ...);

DELETE FROM customers WHERE id = 42;  -- fails: FK constraint violation
```
The FK's job is **referential integrity**: guarantee an `orders` row can never point at a `customer_id` that doesn't exist — not query convenience. Deleting child rows first, then the parent, is the right mechanical pattern for plain cleanup — but it's the wrong call when the child data is financial/legal/audit data that must be retained (e.g. a GDPR "delete my account" request on an order history table). The standard pattern there is the reverse: keep the orders, **anonymize the customer's PII** instead, or repoint `orders.customer_id` at a placeholder via `ON DELETE SET NULL`. Declarative options worth knowing: `ON DELETE CASCADE` (auto-delete children — convenient but dangerous), `ON DELETE SET NULL`, `ON DELETE RESTRICT` (the default-like blocking behavior).

**OFFSET pagination cost.**
```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 25 OFFSET 100000;   -- slow
SELECT * FROM orders ORDER BY created_at DESC LIMIT 25 OFFSET 0;        -- instant
```
Even with an index on `created_at`, this shows as an **Index Scan**, not a full/sequential scan — it's not "the index doesn't work." A B-tree supports jumping to a specific *key value*, not to the *Nth row by ordinal position*. To satisfy `OFFSET 100000`, the engine walks the index entry-by-entry and discards the first 100,000 rows — an `O(offset)` cost inherent to offset pagination. Fix: **keyset/cursor pagination** — `WHERE created_at < <last_seen_value> ORDER BY created_at DESC LIMIT 25`, storing the last row's value as the next page's cursor. If the cursor column isn't unique, use a **compound cursor** `(created_at, id)` with a matching composite index so the tie-breaker makes every row's position unique.

**Soft delete vs. hard delete.**
```sql
-- Hard delete: unrecoverable
DELETE FROM orders WHERE id = ?;

-- Soft delete: reversible
UPDATE orders SET status = 'CANCELLED' WHERE id = ?;
```
Two trade-offs beyond "table growth" (which a partial index on active statuses mostly neutralizes):
1. Every query against the table must remember to filter out soft-deleted rows (`WHERE status != 'CANCELLED'`) — miss it once in a report or join and cancelled rows silently reappear as active.
2. Overwriting a shared lifecycle status column on "delete" can destroy the prior state needed for an accurate restore (an order might have been `PAID`, not a fixed reset value). A separate nullable `cancelled_at` timestamp alongside the real status avoids this — cancelling sets the timestamp without touching status, restoring just clears it.

**Also worth reviewing:** normalization (1NF–3NF) vs. deliberate denormalization (and being explicit about the write-cost accepted); a full status-history table when a durable audit trail of every past transition is needed, not just the current state.

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
What actually changes:
- **`final` + fail-fast**: a missing bean fails application *startup* immediately with a clear error. Field injection can leave a `null` field that only blows up as an NPE later, at the first call site.
- **Plain-JUnit testability**: `new OrderService(mockOrderRepository)` needs zero Spring context. Field injection generally forces `@SpringBootTest` or reflection-based mocking.
- **SRP visibility**: a 7-parameter constructor is an obvious smell; field injection hides that same smell.
- Not about bean lifecycle — Spring manages both as singletons either way.

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
Rule: `@Transactional` rolls back on **unchecked exceptions and `Error`s** by default — checked exceptions need `rollbackFor` explicitly. It also only protects atomicity within **one method's** transaction — it does nothing to coordinate between two *separate* requests/transactions (that's a lost-update problem, solved by locking — see Concurrency).

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
`@Transactional` doesn't work on the method itself — Spring wraps the **bean** in a dynamic proxy that intercepts calls to open/commit/roll back a transaction. That interception only happens when the call arrives **from outside the bean, through the proxy**. `this.saveOrder(order)` is a plain Java call on the raw object — it never touches the proxy, so no transaction opens at all. Fix: extract the method into a **separate injected bean** so the call comes from outside the class and passes through the proxy.
```java
@Service
public class OrderPersistenceService {
    @Transactional
    public void saveOrder(Order order) { ... }
}
```
Reflex: any time `@Transactional` "isn't working," first ask *"is this method self-invoked — called via `this.method()` or a bare unqualified call from another method in the same class?"*

**N+1 problem.**
```java
List<Order> orders = orderRepository.findAll();       // 1 query
for (Order o : orders) {
    o.getCustomer().getName();                          // N additional queries — one per order
}
```
Root cause: **lazy loading** — the `@ManyToOne`/`@OneToMany` association only fires its query when accessed, once per row. Spot it in logs: one query for the parent list, then N nearly-identical queries for a related entity. Fixes: `JOIN FETCH` in JPQL, `@EntityGraph`, or batch fetching (`@BatchSize`) — all collapse it to 1-2 queries.

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
A transient `Customer` (`id == null`) placed in a `HashSet`/used as a `HashMap` key before being persisted will have its hashCode change the instant Hibernate assigns a real `id` on save — the same mutable-hashCode-key problem as any other mutable map key, now on an entity's transient→persistent transition. Fix: avoid blanket `@Data` on `@Entity`; use `@Getter`/`@Setter` plus `@EqualsAndHashCode(of = "id")` (or hand-written equals/hashCode on `id` with a null-safe check). Secondary risk: an all-fields `toString()`/`equals()` can cause infinite recursion if a bidirectional relationship exists. Separately, JPA requires a no-args constructor on `@Entity` classes for reflection-based instantiation — note that Lombok's `@Data` already includes an implicit `@RequiredArgsConstructor`, which is a no-args constructor *as long as there are no `final`/`@NonNull` fields*; it only becomes a real gap once such a field is added.

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
The `Session` a lazy proxy needs to fetch through only lives as long as the `@Transactional` method's transaction. The moment that method returns, the transaction commits, the Session closes, and the returned entity is **detached**. Any association not already initialized *inside* the transaction can never be initialized afterward — accessing a non-id field on it throws (accessing just the FK scalar, e.g. `getCustomer().getId()`, is safe even detached, since JPA can supply the id without touching the DB). Fixes:
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
Anti-pattern to know by name: **"Open Session in View"** — extending the Hibernate session across the whole HTTP request masks this exception at the cost of hidden N+1 queries at render time; Spring recommends `spring.jpa.open-in-view=false` and fixing the real cause instead.

---

## 5. REST & API Design

**Idempotency — "if I fire this exact request twice, is the end state different?"**

| Method | Idempotent? | Why |
|---|---|---|
| `GET` | Yes | Read-only, no state change |
| `PUT` | Yes (by contract) | Setting the same full state twice yields the same end state |
| `DELETE` | Yes (by contract) | Deleting an already-deleted resource is still "gone" |
| `POST` | No (by default) | Usually *creates* something — firing it twice can create two somethings |

```java
// Cheap state-check guard for a PUT-like transition with a side effect
if (order.getStatus() == OrderStatus.CANCELLED) {
    return;  // already cancelled, no-op — makes a retried request safe
}
order.setStatus(OrderStatus.CANCELLED);
paymentGateway.refund(order);
```
Idempotency **keys** are the heavier tool, needed when state alone can't tell you "have I already processed this exact request" — client generates a unique key per logical operation, server stores "already processed this key" and returns the original result on retry without re-running side effects (the pattern Stripe and other payment APIs use).

**Action endpoint vs. generic PUT.**
```
POST /api/orders/{id}/cancel     -- explicit: signals a real action with side effects
PUT  /api/orders/{id}            -- generic: hides that cancelling also refunds/releases inventory
```
A `PUT` can technically be made idempotent too (state-check before re-running the side effect) — the real decision driver is clarity of intent.

**Pagination.**
```
GET /api/v1/books                 -- fine at 500 rows, breaks at 2,000,000
GET /api/v1/books?page=0&size=25  -- bounded response size regardless of table growth
```
Fix belongs at both layers: the SQL query needs `LIMIT`/`OFFSET` or cursor pagination, and the API contract needs `page`/`size` (or cursor) query params.

**DTOs vs. entities.** Returning an `@Entity` directly from a `@RestController` couples the DB schema to the public API contract (a schema change becomes a breaking API change), can leak internal fields (password hashes, flags), and can trigger `LazyInitializationException` when serializing an uninitialized lazy association outside a transaction. A separate DTO/response record decouples the two.

**Status-code precision under a multi-outcome scenario.**
```java
// success                  -> 200 OK
// order id not found       -> 404 Not Found
// order already shipped    -> 409 Conflict   <- NOT 400
```
`400` means the request itself is malformed; `409` means the request is well-formed but the *current state* of the resource conflicts with the action — the correct code for "already shipped, can't cancel." Pair this with a single consistent error envelope across every endpoint (normally via a global `@ControllerAdvice`/`@ExceptionHandler` mapping specific exceptions to status + body), plus a stable, machine-readable error **code** (`"code": "ORDER_ALREADY_SHIPPED"`) alongside the human message, so a frontend can safely switch on the code rather than pattern-matching message text.

**A `POST` that creates a resource — 200 vs. 201.**
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
`200` just means the request succeeded; `201` means it succeeded *and created a new resource*, which is the more precise signal a client actually wants from a creation endpoint.

**400 vs. 422 vs. 409 — three different reasons a request can fail.**
```
400 Bad Request          -- the request itself is malformed (missing field, wrong type, invalid JSON)
422 Unprocessable Entity  -- well-formed, but fails a business/semantic rule (an order with zero items)
409 Conflict              -- well-formed and valid, but conflicts with the resource's CURRENT STATE
```
Reflex question: *"Is something structurally wrong with the request (400), is it well-formed but breaking a rule (422), or is the request fine but the resource's current state won't allow it (409)?"* A common junior mistake is reaching for `409` on a plain validation failure like "order has no items" — that's `400`/`422`, since there's no existing resource state being conflicted with; `409` is reserved for things like "already shipped, can't cancel."

**Resource modeling — path vs. query parameters.**
```
GET /customers/42/orders             -- one customer's orders: id in the PATH (scopes the collection)
GET /customers/42/orders?since=2026-07-28  -- query param filters WITHIN that scope

GET /orders?since=2026-07-28         -- "all customers": no single scope, route reverts to flat + query filter
```
General rule: **path parameters identify which specific resource/sub-collection you're scoped to; query parameters filter or modify the result set within that scope.** Prefer self-documenting query param names (`?since=2026-07-28` or `?days=7`) over custom duration syntax (`?range=7D`).

**Also worth reviewing:** API versioning a field change without breaking existing clients; Jackson basics (`@JsonIgnoreProperties(ignoreUnknown = true)`, date format configuration).

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
Name the concept explicitly: this is a **race condition** — specifically a **lost update**.

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
A plain `HashMap` gives **no thread-safety guarantee whatsoever** — `put()` can trigger an internal resize/rehash of its bucket array, and two threads calling `put()` at the same instant race on that same internal array, silently dropping/overwriting entries or returning `null` for a key that should exist (this is also what throws `ConcurrentModificationException` in the cases that do get caught — a fail-fast check on an internal modification counter). This is a structural-corruption bug, not a cache-staleness problem, so cache invalidation doesn't fix it. Idiomatic fix — `ConcurrentHashMap`, purpose-built for concurrent access:
```java
private final Map<Long, Product> cache = new ConcurrentHashMap<>();

public Product getProduct(Long id) {
    return cache.computeIfAbsent(id, key -> productRepository.findById(key).orElseThrow());
}
```
`computeIfAbsent` also fixes the check-then-act race (`containsKey` → `get` → `put`), making the lookup-or-compute atomic per key. Reflex: any mutable `Map`/`List`/`Set` field in a Spring singleton bean touched by multiple request threads → reach for the matching `java.util.concurrent` type before hand-rolling a `synchronized` block.

**Telling apart the concurrency problem shapes — the highest-leverage distinction to internalize:**

| Shape | Symptom | Correct fix | Wrong fix |
|---|---|---|---|
| **In-process race** — one shared mutable field, multiple threads, same JVM | Lost updates on a counter under concurrent load | `AtomicInteger` / `synchronized` | — |
| **Lost update across two transactions** — two separate HTTP requests each read-then-write the same DB row | One request's edit silently overwritten by another's save later | Optimistic locking (`@Version`) or pessimistic locking (`SELECT ... FOR UPDATE`) | `@Transactional` alone — wraps one method, doesn't coordinate two separate transactions |
| **Duplicate-insert race** — two requests both pass a "does this exist?" check before either inserts | Two rows with the same email after concurrent signups | DB-level `UNIQUE` constraint (no lock needed — nothing to lock yet) | Pessimistic locking — there's no existing row to `SELECT ... FOR UPDATE` |
| **Deadlock via lock-order inversion** — two transactions each pessimistically lock the same two+ rows, but in opposite orders | Both transactions error with a real DB deadlock exception, near-instantly | Enforce one consistent lock-acquisition order across every code path that locks those rows | Explaining it as ordinary lock contention/timeout, or switching wholesale to optimistic locking |

Ask first, every time: **"Is this racing inside one process (needs `synchronized`/atomics), or racing across time via the database (needs a schema constraint or transactional locking)?"**

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
Fits read-heavy, low-conflict situations — no blocking, cheap, but requires handling the conflict exception.

**Pessimistic locking.**
```sql
BEGIN;
SELECT * FROM seats WHERE id = ? FOR UPDATE;   -- row-level lock; other transactions
                                                 -- locking this row must WAIT
-- ... book the seat ...
COMMIT;                                          -- lock released
```
Fits short, high-conflict operations (e.g. last-seat booking) where losing the conflict is expensive — trades throughput for safety, can cause timeouts under heavy load, and introduces deadlock risk if two transactions lock multiple rows in different orders. Keep the transaction short.

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
A **deadlock** is a circular wait: each transaction holds what the other needs next, forming a cycle with no possible progress. It is *not* the same as ordinary lock contention (many transactions serialized waiting in one direction, which eventually resolves and is never flagged as a deadlock). A database's deadlock detector finds a real deadlock almost immediately (milliseconds, not a load-dependent timeout) and kills one transaction to break the cycle. Fix: enforce **one consistent lock-acquisition order** across every code path that touches the same rows (always lock `orders` before `inventory`, or lock by ascending row ID) — not a wholesale switch to optimistic locking.

**Distributed locking.** A `synchronized` lock is tied to an object living inside **one JVM process**. Two app instances are two separate JVMs, each with its own independent copy of that lock object — neither has visibility into the other's lock, so two threads across instances can both enter a "synchronized" method at the same instant. In-process tools (`synchronized`, `AtomicInteger`) only ever coordinate threads *within a single process* — once there's more than one instance, they stop working entirely, silently. The Redis `SET key value NX PX <ttl>` pattern (set-if-not-exists with an expiry) is the basic idea for a cross-instance lock — Redis becomes the one coordinator every instance defers to. Often you don't need a lock at all: a unique constraint, or an atomic `UPDATE ... WHERE version = ?`, already gives the guarantee needed for "the same request might get delivered twice."

---

## 7. Testing & Debugging

**Debugging an intermittent, timing-dependent bug — the sequence matters:**
1. **Reproduce reliably first.** A debugger is close to useless until the bug can be triggered on demand. If the report includes a trigger ("double-click," "fast," "sometimes"), recreate it mechanically (script two near-simultaneous requests) rather than clicking quickly by hand.
2. **Be deliberate about tool choice for timing bugs.** A debugger breakpoint changes the timing of everything around it, which can make a race condition disappear (a "Heisenbug"). Logging observes without altering timing, and can run in production where the bug actually happens.
3. **Use structured logging, not `println` + redeploy.** A real logger has levels (`DEBUG`/`TRACE`) and structured context — request ID, timestamp, user/order ID, thread/instance ID — so a specific failing request can be isolated.
4. **Form an explicit hypothesis for *why* it's intermittent** before reaching for a tool. Candidate hypotheses for a "sometimes" bug: a non-idempotent endpoint hit twice (double-submit), an exception silently swallowed inside a loop, a job running on two instances/threads simultaneously.
5. **Binary-search debugging** once you can reproduce: narrow down *where* in the flow the bug is introduced by checking state at successively narrower points.

**Not every intermittent bug is timing-based — resource/threshold effects need a different hypothesis.**
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
A deep category tree throwing `StackOverflowError` intermittently, with **identical input** each time, is not a logic bug or a timing race — it's a **resource/threshold bug**. The recursion depth sits right at the edge of the JVM's available stack space; small, incidental variations in how much stack the surrounding call chain already consumed (which pooled thread handled the request, how many filter/proxy layers ran first) tip it over the edge on some runs and not others. Confirm by logging the recursion depth reached and comparing against the JVM's default stack size — don't just assume "the tree is too deep" without asking why it's *sometimes* too deep. Fix: convert the recursion to an **iterative** traversal using an explicit stack (`Deque<Category>`), removing the dependency on JVM stack size entirely. Raising `-Xss` only raises the threshold, it doesn't remove the fragility.

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
Every collaborator is mocked, and every mock returns exactly the value the assertion checks. The tell: if you deleted `DiscountService`'s real implementation and replaced it with `return 90.0;`, this test would still pass — a test that survives deleting the logic it's meant to verify isn't testing that logic. Fixes: add `verify(mockPricing).getBasePrice(product)` to confirm orchestration (the service calls collaborators with the right arguments), or recognize that mocking every dependency is itself the smell and test the real math against a real/minimally-faked collaborator. Reflex: if every dependency is mocked *and* the stub's return value matches the assertion, ask "what real behavior is left to fail if I break something?"

**Test-driving a bug fix.** Before fixing a reported bug, don't just "check/run the existing tests" — if a bug reached production, there is almost certainly **no existing test** covering that exact case (otherwise CI would have caught it). Correct sequence:
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

**Flaky tests, weak assertions, one-behavior-per-test.**
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
1. **One test, three behaviors.** A vague method name and multiple unrelated assertions mean a failure requires reading the stack trace closely to know what broke. Fix: one behavior per test, named for that behavior.
2. **A loose assertion doesn't test what it claims to.** `assertTrue(total < 200.0)` would pass even if the coupon did nothing. Fix: assert the exact expected value with a small tolerance — `assertEquals(135.0, order.getTotal(), 0.001)`.
3. **A flaky assertion with zero code changes and no local repro is a test-isolation smell, not a coincidence.** Likely cause: shared mutable state leaking across tests (a static/shared fixture reused without resetting, a test DB not rolled back, parallel execution touching shared state). Fix: find and eliminate the shared state (fresh data per test, `@BeforeEach` reset) — never loosen the assertion or re-run until green.

Reflex for "test fails sometimes, no code changes, can't repro locally": ask *"is this test's setup actually isolated?"* before assuming a timing race — plain shared-fixture bugs are usually about isolation, not concurrency.

**Unit vs. integration tests — the real distinguishing question.**
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
The distinguishing lens isn't "how much business logic" a method has (almost every method has some) — it's **what boundary is actually being verified**: pure logic in isolation (mock every collaborator) → unit test; a real interaction across a boundary (does this query actually return the right rows against a real schema) → integration test. The **testing pyramid**: many fast, isolated unit tests at the base, fewer integration tests validating real boundaries, very few end-to-end tests at the top. Ask of any test: "which collaborators am I trusting vs. which one am I actually verifying?" Everything trusted gets mocked; only the thing under test runs for real.

---

## 8. Web Security & Frontend Basics

**CORS — full mechanism.**
```
localhost:3000 (frontend)  ---fetch()--->  localhost:8080 (backend)
```
Different **port** already means a different **origin** (scheme + host + port all must match), triggering the browser's **same-origin policy**. Critical nuance: **the request still reaches the backend and is processed** — what the browser blocks is letting frontend JavaScript *read the response*, unless the server's headers explicitly allow it.
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
**Preflight `OPTIONS` request**: for "non-simple" requests (JSON body, custom headers, methods beyond simple GET/POST-form), the browser first sends an automatic `OPTIONS` request asking permission. If the backend doesn't answer correctly, the browser never sends the real request at all — this is why a blocked call can show **zero log lines** in the controller. Production gotcha: `allowedOrigins("*")` combined with `allowCredentials(true)` is rejected by browsers outright — use an explicit domain whitelist in production.

**Request lifecycle — typing a URL and hitting Enter:**
1. **DNS resolution** — domain name → IP (browser cache → OS cache → resolver → root/TLD/authoritative servers).
2. **TCP handshake** — SYN → SYN-ACK → ACK, establishing a reliable connection before any data is exchanged.
3. **TLS handshake** (if HTTPS) — certificate exchange/verification, encryption keys negotiated, before any HTTP request is sent.
4. **HTTP request/response** — browser sends the request line + headers, server responds, browser parses and renders.

`ERR_CONNECTION_REFUSED` happens at step 2 — the TCP handshake was actively rejected (nothing listening, service down, firewall). This is fundamentally different from a *timeout*, which means the packet went unanswered rather than being refused. DevTools → Network tab → "Timing" breakdown shows the real phases with real numbers.

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
A malicious `name` value could break out of the string literal and inject additional SQL (e.g. a `JOIN`-based attack to exfiltrate data from unrelated tables). Why `PreparedStatement` fixes it: the query structure is compiled by the database *before* the value is bound, so user input is never re-parsed as SQL syntax, regardless of what characters (`'`, `;`, `--`) it contains. Server-side input validation is fine as defense-in-depth but is **not** a substitute — blacklist-style validation is routinely bypassed.

**Cookie tampering vs. JWT signing.**
```
// Insecure: user=John;role=admin  stored directly in a plain cookie
```
The sharpest risk with storing role/identity in a plain cookie isn't primarily an outside attacker reading it (confidentiality) — it's that the *same logged-in user* can open DevTools and edit `role=admin` onto their own cookie to self-escalate privileges (an **integrity/tampering** problem, no outside attacker needed). A JWT fixes this because it is **cryptographically signed** (HMAC/RSA): if a client edits any claim without the server's secret key, the signature no longer matches on verification, and the request is rejected outright — "attach it to a header and validate server-side" alone isn't the mechanism; the signature check is. The sibling standard answer is a traditional **server-side session** — the cookie holds only an opaque random session ID, and the actual role/user data lives server-side (DB/Redis), looked up per request. Trade-off: JWT avoids a per-request lookup but can't be revoked instantly; a session ID is trivially revocable but costs a lookup.

**Session cookies vs. JWTs — stateful vs. stateless is the first-level answer, but not what actually decides it.**
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
"Stateful vs. stateless" is correct as a label, but the two properties that actually drive the choice in practice are **revocability** (can you kill it server-side right now, e.g. for "log out everywhere" or a compromised account?) and **theft-resistance** (where does the token sit, and can injected JS read it?). Neither scheme is inherently "better" — a backend can run both at once (session for the first-party web app, JWT for mobile/other services) via separate auth filters; that's routine, not a conflict.

**Also worth reviewing:** never trust client-side-only validation (hiding a button isn't security — always re-validate server-side).

---

## 9. System Design & Architecture Basics

**Layered architecture.** Controller (API surface) → Service (business logic) → Repository (persistence), with clear ownership boundaries at each layer.

**Caching — cache-aside pattern.**
```
1st request:  client -> app -> (cache MISS) -> DB -> app writes to cache -> client
2nd request:  client -> app -> (cache HIT)  -> client   (DB never touched)
```
A too-long TTL risks overselling on fast-changing data (cached "5 in stock" goes stale while real stock hits 0 — a `quantity < 0` guard is a necessary safety net). Vocabulary:
- **Cache stampede / thundering herd** — the too-short-TTL failure mode: many requests miss the cache at the exact same instant (a hot key expiring) and all hit the DB simultaneously.
- **Invalidate-on-write** — the complement to TTL: the moment stock/price actually changes, evict/update the cache entry immediately rather than waiting for TTL expiry.
- Rule of thumb: long TTL for slow-changing fields (description, images), short-TTL-or-invalidate-on-write for fast-changing/high-stakes fields (stock, price).

**Bottleneck reasoning.** The reliable heuristic: **"which operation happens far more often, reads or writes?"** For a URL shortener, redirect/lookup traffic vastly outweighs create traffic (often 100:1+), so the bottleneck is the read/redirect path — fix with a cache (e.g. Redis) in front of the DB for hot short codes, plus a unique index on `short_code` so lookup itself isn't a table scan. Redirect mechanics: HTTP `301` (permanent) or `302` (temporary) with a `Location` header.

**CRUD schema design checklist.** Walk every planned endpoint against the schema before finalizing it: **list, get-one, create, update, delete** — does the schema have what each of those five needs to read or write? (Common misses: no `completed` boolean for a "mark complete" API; no list/search endpoint at all.)

**Sync (direct call) vs. async (message queue).**
```
Direct call (sync OR async style)          Message queue
caller ---request---> callee               caller --publish("OrderPlaced")--> queue ---> consumer ---> email service
caller still needs callee's result          caller returns immediately, never waits on
or has to handle its failure right now      the consumer or downstream service at all
```
`async/await` or a non-blocking `HttpClient` call is still a **direct call** — it just doesn't block the calling thread; the caller is still coupled to that service's availability. A **message queue** (RabbitMQ/Kafka/SQS) is a stronger, different kind of decoupling: the caller publishes and returns immediately, a separate consumer processes it whenever it can (even if the downstream service is down), and retries on transient failures come for free. Decision rule: **"Does the caller need the result before it can respond to the user?"** Payment → yes, direct call. Confirmation email → no, queue candidate.

**Stateless services.**
```
1 instance:              login -> writes session to in-process Map -> works fine
2 instances, round-robin: login -> server A's Map -> next request routed to server B -> B's Map has nothing -> "not logged in"
```
General principle: services should be **stateless** — anything living in a singleton/in-process field (a session map, an in-memory cache, a rate-limiter counter) only exists on the instance that wrote it, and silently breaks the moment there's more than one instance. Standard fix: externalize the store to Redis (or a shared DB) so every instance reads/writes the same source of truth. Alternative: **sticky sessions** (load balancer routes the same client to the same instance via cookie/IP hash) — simpler to bolt on, but fragile (a restarted instance loses every session pinned to it) and can unbalance load.

---

## 10. Tooling: Git, Build & Environment

**Commit hygiene.**
```
fix: reject empty bearer token in auth middleware
chore: rename OrderProcessor -> OrderPipeline
refactor: reformat build.gradle
```
Split a mixed commit by logical concern using conventional-commit prefixes. Why it matters beyond style:
- **`git blame`/`git bisect` clarity** — a future regression search should land on a commit whose message actually describes what changed there, not "fixes stuff" that also bundles two unrelated changes.
- **Revert safety** — if one change turns out wrong and needs reverting, a bundled commit forces reverting unrelated changes along with it.
- **Review noise/signal** — a security-sensitive change sitting next to cosmetic renames is easy for a reviewer to skim past by accident.

Reflex: *"If I had to revert or `git blame` just this one change in three months, could I isolate it cleanly?"* If not, split the commit — `git add -p` splits hunks within a single file.

**Secrets in git.** A committed secret **stays in git history forever**, even after a later commit deletes the file — anyone with clone access can still find it in an old commit, which is why rotation is mandatory once a secret has been pushed even once. Add `.env` to `.gitignore` immediately, commit a `.env.example` with placeholder keys/values so onboarding still works. Real secrets belong in a runtime-injected source — CI/CD secret store or a secret manager (Vault, AWS Secrets Manager) — never hardcoded in the repo, even a "local-only" file that gets committed by accident. Bonus habit: secret-scanning in CI (`gitleaks`, GitHub secret scanning) catches this before merge.

**Maven/Gradle dependency conflicts.**
```bash
mvn dependency:tree            # prints the resolved graph, flags dropped versions as "(conflict)"
./gradlew dependencies         # Gradle equivalent
./gradlew dependencyInsight --dependency jackson-databind
```
Maven's resolution rule is **nearest-wins**: the declaration closest to your own project in the tree wins, regardless of which version is actually newer — a transitive dependency three levels deep can silently lose to an older version declared one level up. Fixes: pin the version explicitly in your own `pom.xml` (a direct declaration always wins over transitive), use `<dependencyManagement>` to centralize versions across a multi-module project, or `<exclude>` the unwanted transitive version.

**Resolving a merge conflict — combine both, don't pick one side.**
```
<<<<<<< HEAD
// your current branch's version
=======
// incoming version from main
>>>>>>> main
```
If a teammate restructured a method and you separately added a new field/behavior inside it, the correct resolution usually has **both** changes — not one or the other; picking a side wholesale is how real work quietly disappears. Process: read both blocks, manually combine into one correct version, delete the markers, **test/run it** (a syntactically clean merge can still be semantically wrong), then `git add`/`git commit` (merge) or `git rebase --continue` (rebase). An IDE's 3-way merge view makes "combine both" more obvious than eyeballing raw markers. `git log -p <file>` or `git blame` on the conflicting lines often shows exactly what each side changed before you even need to ask a teammate.

**Environment config across dev/staging/prod.** A `Could not resolve placeholder 'DB_PASSWORD'` failure in staging (but not locally) usually means something local-only (a `.env` file, an IDE run config, an exported shell variable) supplied the value on the laptop, and it was simply never set up in staging — implicit local machine state that didn't travel with the app. Two-lens fix:
- **Config that differs by environment but isn't secret** → Spring profiles: `application-dev.properties` / `application-staging.properties` / `application-prod.properties`, switched via `spring.profiles.active=<profile>`.
- **Actual secrets** → platform-injected runtime environment variables (CI/CD secret store, Kubernetes Secret, AWS Secrets Manager) — never a repo file, even a per-environment one. `.env` is fine for local dev convenience but must be `.gitignore`'d the moment it holds a real secret; commit a `.env.example` with placeholders instead.

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
`merge` leaves existing commits untouched and adds a new two-parent merge commit — non-linear, nothing rewritten. `rebase` replays your commits on top of `main`'s current tip one at a time — same content, but every commit gets a **brand-new hash**, producing a linear history. Both merge and rebase preserve each commit's original **author metadata identically** — `git blame`/`git log --author` works the same either way; rebase doesn't affect authorship, only the commit hash. The real reasons to prefer rebase are a cleaner linear history and an easier-to-read PR diff.

**Rule to memorize: never rebase (and force-push) a branch other people have already pulled and built on.** Because rebase mints new hashes for every commit, a teammate who already pulled your branch and committed on top of the *old* hashes is now diverged at the hash level — force-pushing pulls the rug out from under their local branch (their commits' parents no longer exist upstream), risking lost or duplicated work. This is a history-rewrite problem, not an ordinary merge conflict. Rebase freely on a private branch only you use; once it's shared, prefer merge, or coordinate the rebase explicitly with everyone who has it.

**Responding to PR review feedback you disagree with — evidence over assertion.**
1. Check the code, not your memory — trace whether the flagged concern (e.g. a second code path) actually calls the same logic you changed, or a separate implementation.
2. Turn the check into evidence, not just a claim — point to the specific file/line so the reviewer can verify it themselves.
3. Better still, write a quick test exercising the questioned path with the input that triggered the original issue — concrete proof, and a permanent regression guard either way.
4. If you turn out to be wrong, this same process surfaces that *before* you argue in the thread, not after.

Reflex: a code-review disagreement is resolved with evidence (code, tests, logs) — never by restating your own confidence level.
