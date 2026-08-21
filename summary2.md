# Backend Interview Knowledge Reference — Sessions 2026-08-12, 2026-08-17, 2026-08-19, 2026-08-20 & 2026-08-21

A pure study reference distilled from these mock interview sessions — concepts only, organized by topic, with code examples where they clarify the mechanism.

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
Because `String` is immutable, `csv += ...` can't mutate the existing object — the compiler rewrites it roughly as `csv = new StringBuilder(csv).append(...).toString()`. That means every single iteration allocates a *brand-new* `StringBuilder`, copies the *entire* string accumulated so far into it, appends the new piece, then converts back to an immutable `String`. Iteration *i* copies `i` characters, so total characters copied across `n` iterations grows quadratically — O(n²) total work, not O(n) — which is exactly why a "simple" loop looks instant at 20 rows and becomes the profiler's top hot spot at 50,000.

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
Reflex: any `+=` building a `String` inside a loop means "a new object gets allocated and everything so far gets recopied, every iteration" — replace it with a single `StringBuilder` declared outside the loop.

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
It "feels like one responsibility" because everything happens to touch a `User` — but SRP isn't about the noun, it's about *reasons to change*. This class changes when signup validation rules change, when marketing edits the welcome email copy, or when the analytics team adds a field to the login event — three unrelated teams/reasons touching the same file. Concrete costs: testing `createUser`'s validation logic drags in mocking email delivery even though the test has nothing to do with email; reusing "send welcome email" from a different signup path (e.g. an admin bulk-import script) means duplicating the call or routing an unrelated flow through `createUser`.

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
Extracting a class fixes SRP but not DIP by itself — if `UserService` still takes a concrete `EmailService` as its dependency, swapping to push notifications or SMS later still means editing `UserService`. Injecting the *interface type* (`NotificationSender`) is what lets the concrete implementation change — or a second implementation be added — without touching the class that uses it. In Spring: the concrete class is `@Service`-annotated and implements the interface; the consumer declares the interface type in its constructor, and the container wires the concrete bean automatically.

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
Today the app only ever sends email — no SMS feature, no config, no ticket for one. The interface-plus-factory scaffolding was built for a variation that doesn't exist yet, only *might* exist someday. That costs something now: an extra interface, an extra class, a factory lookup to trace through — for flexibility nothing currently uses.
```java
// Simpler, and equally correct until a second real implementation actually needs to ship
class EmailSender {
    void send(Order order) { /* send email */ }
}
```
The signal to watch for isn't a class count — it's whether a **second real, active implementation** exists or is committed, scheduled work, not "we might want this later." Introduce the `Strategy`/`Factory` split when that second implementation actually needs to ship. Separately, a branch like `SmsNotificationStrategy` that nothing ever calls is dead code and worth flagging for removal regardless of the abstraction question.

---

## 3. Databases: SQL, Indexing & Data Modeling

**A single FK column models one-to-many — belongs-to-*multiple* needs a junction table, not more columns.**
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
The moment a real requirement is "a product belongs to *multiple* categories," the relationship is many-to-many, and a single FK column on either side can't express that — there's no fixed number of columns that's ever really enough. The standard fix is a **junction table** (a.k.a. join table): a table whose only job is to record which pairs are linked.
```sql
CREATE TABLE product_categories (
  product_id BIGINT REFERENCES products(id),
  category_id BIGINT REFERENCES categories(id),
  PRIMARY KEY (product_id, category_id)
);
-- category_id comes OFF the products table entirely — categories now live only here
```
The two FK columns together form a **composite primary key**, which does double duty: it's the row identifier, and it's a free guarantee that the same product can't be tagged into the same category twice (inserting the same pair again violates the PK). Querying "all products in category X" also gets simpler — one join against `product_categories`, instead of an `OR` across a growing list of columns.

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
`orphanRemoval = true` tells Hibernate: "if an `OrderItem` is ever removed from this collection, it no longer belongs to any parent — delete it from the database." `order.getItems()` returns the real, Hibernate-managed collection, not a copy — calling `.remove()` on it mutates the actual tracked association. Hibernate's dirty-checking notices the change at flush time (end of the transaction) and issues a genuine `DELETE`, exactly as if `repository.delete(item)` had been called directly. The danger is intent mismatch: code that only meant to filter a collection for *display* purposes has no way to signal "this isn't a real removal" — any mutation on a managed collection inside an open transaction is a real write, whether or not that was the intent.

**Misconception to name explicitly: a field initializer does not protect a loaded entity.**
```java
private List<OrderItem> items = new ArrayList<>();
```
This initializer only ever runs when constructing a brand-new, transient instance via `new Order()`. When Hibernate loads an *existing* row, it replaces the field with its own managed collection wrapper (a `PersistentBag`) during hydration — the initializer's value is simply discarded at that point, not merged with or protecting the real data.

Fix: never call mutating collection methods (`remove`/`add`/`clear`) on a live, managed entity collection from read-only or reporting code — copy it first (`order.getItems().stream().filter(...).toList()`). Reserve `orphanRemoval = true` for cases where "removed from the parent's collection" and "should be deleted" are genuinely the same business rule — not as a default that comes bundled with `cascade = ALL`.

---

## 5. REST & API Design

**A schema-accurate API spec can still be unusable — types tell you the shape, not the exact contract.**
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
An integrating team can read this and still get it wrong in ways that generate real support tickets:
- Is `discountCode` omitted, or sent as `""`? The spec doesn't say whether those behave differently.
- What format does a date field expect — `2026-08-12`, an ISO-8601 timestamp, a Unix epoch?
- What does a *valid* `OrderItem` actually look like nested inside `items` — is there a minimum count, what if one item carries its own discount?

None of that is a documentation *error* — the schema is accurate. It's a documentation *gap*: a type system describes shape, not the concrete values that satisfy real use cases.

**Unbounded list endpoints eventually break — and `LIMIT` alone hides data instead of fixing that.**
```sql
-- The "fix" that isn't one: stops the timeout, but nothing lets a client ever ask for row 101+
SELECT * FROM orders WHERE customer_id = ? LIMIT 100;
```
This bounds the response, but there's no `page`/`offset` or cursor parameter anywhere in the API — the rest of that customer's 40,000 rows aren't slow to reach anymore, they're unreachable. It has a second, subtler bug: without an `ORDER BY`, SQL makes no guarantee about *which* 100 rows come back, so "the first 100" can differ between otherwise-identical calls.

```sql
-- Offset pagination: simple, but cost grows with depth
SELECT * FROM orders WHERE customer_id = ? ORDER BY id LIMIT 100 OFFSET 39900;
-- the DB scans and discards 39,900 rows before it can return the 100 you actually asked for

-- Cursor (keyset) pagination: flat cost regardless of depth
SELECT * FROM orders WHERE customer_id = ? AND id > :lastSeenId ORDER BY id LIMIT 100;
-- an index seek straight to the row after the cursor — nothing scanned and thrown away
```
`OFFSET n`'s cost grows with `n` because the database still has to walk through and discard every skipped row first. A cursor anchored on an indexed, stable column (`id`, or `created_at` plus an `id` tiebreaker) seeks directly to where the last page ended — flat performance whether it's page 2 or page 400. The API contract becomes something like `GET /orders?cursor=<lastId>&size=50`, returning the page plus a `nextCursor` for the next call. Reflex: any endpoint that returns "all rows matching X" with no `page`/`size` or `cursor` parameter will eventually break once one row-owner's data outgrows a single response — add pagination before that happens, not after the ticket comes in.

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
A concrete `example`/`examples` block in the OpenAPI spec answers every one of the questions above in one look — a value the integrating developer can copy, run, and see work, rather than infer-and-guess from field types and then debug against 400 responses. Reflex: when a bug report says "the docs are wrong" but the schema checks out field-by-field, the fix is almost never *more explanatory text* — it's a runnable example covering the realistic cases (a plain request, one with an optional field populated, an edge case like a minimal request).

---

## 6. Concurrency & Locking

**`synchronized` protects against overlapping threads at the same instant — it does nothing for a retried request arriving later.**
```java
// Doesn't do what it looks like it does, for this problem:
@PostMapping("/webhooks/payment-confirmed")
public synchronized ResponseEntity<Void> handlePaymentWebhook(@RequestBody PaymentEvent event) {
    orderService.markPaid(event.getOrderId());
    return ResponseEntity.ok().build();
}
```
A payment provider's webhook retry isn't a second thread racing the first one *at the same moment* — it's a brand-new, separate HTTP request that arrives seconds or minutes after the first one already finished and returned. `synchronized` only ever blocks a thread from entering a block while another thread is *currently inside it*; once the first request has completed, there's nothing left for `synchronized` to block against. The bug this is meant to catch — the same payment event being processed twice — sails straight through.

**When a duplicate is a matter of identity, not timing, the fix is a uniqueness guarantee, not a lock at all.**
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
The webhook payload carries a unique event ID exactly so a handler can atomically check-and-record "have I seen this before?" A `UNIQUE`/primary-key constraint on `event_id` makes that check atomic at the database level, regardless of how many requests hit it or how far apart in time they arrive — no `synchronized`, no distributed lock, no extra infrastructure required. Reflex: separate "two requests at the *same instant*" (a concurrency problem, needs a lock) from "the *same* request arriving again later" (an identity/idempotency problem, needs a uniqueness guarantee) — they look similar but call for different tools.

**A single-JVM lock (`synchronized`) doesn't coordinate across instances — cross-instance mutual exclusion needs an external, atomic check-and-set.**
```java
// Looks like it should prevent triple-execution — doesn't, once there's more than one instance
@Scheduled(cron = "0 0 2 * * *")
public synchronized void sendAbandonedCartReminders() {
    cartService.emailAbandonedCarts();
}
```
Scaling from one instance to three doesn't multiply threads inside one JVM — it creates three separate JVMs, each with its own thread pool and its own independent `synchronized` lock. All three fire their own 2 AM cron trigger, and each JVM's lock only ever blocks *other threads in that same JVM* — there's nothing in `synchronized` that knows the other two instances exist, so the job still runs three times.

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
`SET key value NX PX <ttl>` succeeds only if the key doesn't already exist, and that check-and-set happens as a single atomic operation — no window where two instances could both pass a "does it exist?" check before either writes. The `PX <ttl>` expiry is not an optional extra: without it, an instance that acquires the lock and then crashes mid-job leaves the key set forever, and the job never runs again on any instance. With a TTL, the lock auto-expires even if the holder never releases it explicitly, so the next scheduled run can acquire it fresh. Reflex: whenever `synchronized` is proposed as the fix for "duplicate work across multiple instances/services," the immediate question is "which JVM does this lock live in?" — a lock scoped to one process can never coordinate processes it doesn't share memory with.

---

## 7. Testing & Debugging

**Read the exception message before the frames — it often already names the null and the failed call.**
```
java.lang.NullPointerException: Cannot invoke "String.trim()" because the return value of "com.acme.orders.OrderService.getCustomerEmail(Order)" is null
	at com.acme.orders.NotificationSender.send(NotificationSender.java:42)
	at com.acme.orders.OrderController.checkout(OrderController.java:87)
	at org.springframework.web.method.support.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:150)
	... 40 more lines of Spring internals ...
```
Modern Java's "helpful NullPointerException" messages already spell out what was null and what operation failed on it — `getCustomerEmail(Order)` returned null, and `.trim()` was called on that result. That sentence alone often narrows the search before a single frame is read.

**Skip framework frames on purpose — stop at the top-most frame that's your own code.**
The dozens of `org.springframework.*` lines below the crash are Spring's dispatch machinery calling into the controller, not the bug. Consciously skip any frame whose package isn't yours and stop at the first one that is — here, `NotificationSender.java:42`, called from `OrderController.java:87`. That's the habit that scales: a 5-line trace and a 50-line trace get read the same way, there's just more noise to skip in the second one.

**A null-guard at the crash site fixes the crash, not the bug.**
```java
// Stops the NPE, but doesn't explain anything
if (email != null) {
    email.trim();
}
```
This prevents the exception, but the real question is *why* `getCustomerEmail(Order)` returned null in production — a guest checkout with no email on file, a migration that left old rows without one, a bug upstream in how the order was built? A guard at the call site is a reasonable safety net, but stopping there without asking that second question leaves the same root cause free to resurface anywhere else that touches a null email. In production specifically — unlike local dev — also tie the trace back to the actual order/request that triggered it (via logs or the order ID), since it usually can't just be re-run to reproduce.

**Narrowing down which step failed in a multi-call chain: reach for what's already captured, or a trace, before scattering new print statements.**
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
"Add a `System.out.println` after every line across all five methods and redeploy" turns every guess into a full redeploy cycle, and still only tells you about the *next* failure, not this one. Two better moves, in order:
1. **Check what's already captured first.** The generic `catch (Exception e)` above still logs the real, specific exception (`log.error("Checkout failed", e)`) even though the client only sees a generic message — the answer is very likely already sitting in the logs, searchable by request ID, without deploying anything new.
2. **For a live, hard-to-reproduce-locally failure, reach for tracing over prints.** A distributed tracing tool (e.g. Tempo, viewed through Grafana, correlated with logs in Loki) records a span for each of the five calls in the chain, with timing and status — so instead of guessing which of five methods to instrument, you look at the trace and see directly which span failed or where the chain stopped.

The underlying principle either technique leans on is **binary-search debugging**: narrow the search space by checking a midpoint ("did `reserveInventory` complete? yes → the bug is in the last two steps") rather than instrumenting the whole chain at once and hoping the output makes it obvious. Reflex: before adding any new logging/print statements to hunt a bug, ask "is this already being captured somewhere I haven't looked (existing logs, an existing trace), before I add anything new?"

---

## 8. Web Security & Frontend Basics

**`useEffect` dependency arrays compare by reference, not by value — an inline object/array/function literal breaks that silently.**
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
React compares each dependency-array entry with `Object.is` — reference/identity equality, not a deep/value comparison. `{ include: 'address' }` is structurally identical on every render, but it's a *different object in memory* each time the component function runs, so React sees "changed" every time. The resulting loop: render creates a new `options` → effect sees a new reference → runs `fetchProfile` → `.then(setProfile)` updates state → the state update triggers a re-render → a new `options` object is created again → repeat forever.

```jsx
// Fix: give the dependency a stable reference across renders
const options = useMemo(() => ({ include: 'address' }), []);   // idiomatic — stable identity for a static value

// or, simplest of all when the value truly never changes: hoist it out of the component
const OPTIONS = { include: 'address' };
function ProfilePage({ userId }) { ... }
```
`useState` also happens to "work" here, since the initial value is only constructed once and the reference stays stable unless `setOptions` is explicitly called — but it's not the idiomatic tool, because `useState` signals a value that changes over time, which this isn't. Reflex: whenever an object/array/function literal is defined inside the component body and placed in a dependency array, ask "same reference every render, or a new one?"

**Conditionally rendering a UI element is UX, not authorization — the server has to re-check every time, because the client can always be bypassed.**
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
`{user.role === 'admin' && ...}` only decides whether *this* button renders in *this* browser tab — it has zero effect on whether the backend endpoint it calls will accept a request. Nothing about React, the component tree, or `user.role` runs on the server; a non-admin user (or anyone holding a valid token, admin or not) can call the same endpoint directly with curl, Postman, or the browser's Network-tab replay, and the server will process it exactly as if the button had been clicked — because as far as the server is concerned, no button was ever involved.
```java
// The check has to be re-asserted here, independent of anything the client sent or hid
@PreAuthorize("hasRole('ADMIN')")
@PostMapping("/discounts/employee")
public ResponseEntity<Void> applyEmployeeDiscount(@AuthenticationPrincipal UserDetails user) {
    discountService.applyEmployeeDiscount(user);
    return ResponseEntity.ok().build();
}
```
`@PreAuthorize` checks the role carried in the authenticated principal — derived from a signature-verified JWT claim or a server-side session — which the client cannot forge, unlike a plain `user.role` value living in browser-side JS state. The frontend check is worth keeping (no reason to show a button an employee can't use), but it is UX polish layered *on top of* server-side authorization, never a substitute for it. Reflex: for any sensitive action wired to a UI element, ask "what stops someone from hitting this endpoint directly, skipping the UI entirely?" — if the answer is "nothing," the button hasn't secured anything.

---

## 9. System Design & Architecture Basics

**Caching is a staleness-vs-load trade-off — the acceptable staleness window depends on the cost of being wrong, not just on read volume.**
```
Admin updates price  ---->  DB price = $80
                                          \
Cache still holds     ---->  Cached price = $100   (TTL: 10 minutes)
                                          \
Customer checks out during that window  ---->  charged $100 for something that's actually $80
```
Adding a cache in front of a hot read (`GET /products/{id}/price`) is the right instinct when the DB is struggling under read load — but the TTL chosen implicitly decides how long a wrong value can survive after the source of truth changes. A long TTL for a rarely-changing field (product description, image URL) is a fine trade — mild staleness costs nothing. The same TTL on price data means "for up to 10 minutes, whatever cached number exists might be actively wrong," and if that number gets charged rather than just displayed, staleness stops being a UX nuisance and becomes a financial/legal problem.

**Shortening the TTL narrows the window, it doesn't close it — invalidate-on-write is the complementary fix, not a substitute.**
```java
// Cache-aside with invalidate-on-write: the write path evicts the stale entry
// immediately, instead of waiting for it to expire.
public void updatePrice(Long productId, BigDecimal newPrice) {
    priceRepository.updatePrice(productId, newPrice);
    priceCache.evict(productId);   // next read is a guaranteed cache MISS -> fresh from DB
}
```
Even with a short TTL *and* invalidate-on-write, there's still a race: a request already mid-flight when the price changes can still read the old cached value a moment before eviction happens. For data where being wrong has a real cost, the pattern that closes that gap is to split the read path by purpose: serve the *cached* value for display, but re-verify against the *authoritative* source at the exact moment the value is acted on.
```java
public OrderResult checkout(Order order) {
    // Do NOT trust the cached price here, however fresh it looks.
    BigDecimal authoritativePrice = priceRepository.findCurrentPrice(order.getProductId());
    return paymentGateway.charge(authoritativePrice);
}
```
Rule of thumb: cache freely for reads whose worst case is "the user saw a slightly stale number" (search results, product listings); never let a cached value be the thing that actually gets charged, allocated, or committed — re-fetch from source at that specific instant instead.

**A missing service layer isn't just a style nit — it's four concrete costs, each independent of the others.**
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
A controller's actual job is translating HTTP in and out — parse the request, call the service, map the result to a status code/body. None of that is "loyalty points" or "discount math." Once a business rule lives in the controller, the controller now has two unrelated reasons to change: an HTTP contract change, and a discount-rule change. Four separate, concrete costs follow from that, not just "it's harder to read":
- **Testability** — exercising "does a 1000+ point customer get 10% off" requires spinning up `MockMvc`, a fake `@PathVariable`, a fake `HttpServletRequest` — full MVC machinery just to test a calculation. A plain `OrderService` method is a one-line unit test with none of that.
- **Reusability** — if a batch job, an admin tool, or another endpoint later needs the same discount rule, it's stuck inside this one controller method; a service method is callable from anywhere.
- **Transaction boundaries** — `@Transactional` idiomatically belongs on the service method. As the logic grows (discount + inventory check + audit log), that annotation is what makes it one atomic unit — the controller isn't the natural place for it.
- **Change isolation** — a pure HTTP-layer change (renaming the route, changing the response envelope) and a pure business-rule change (adjusting the loyalty threshold) end up touching the same method, so neither change is isolated from the other.

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
