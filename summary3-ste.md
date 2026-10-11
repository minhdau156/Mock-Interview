# Backend Interview Knowledge Reference — Sessions 2026-09-06, 2026-09-07, 2026-09-13, 2026-09-17 & 2026-10-11

A study reference from these mock interview sessions. It contains concepts only, grouped by topic. Code examples show the mechanism.

---

## 1. Core Java & CS Fundamentals

**Checked vs. unchecked is a class-hierarchy rule. An unclosed stream leaks a file descriptor, not heap memory.** (2026-09-13, Q1, scored 3/10 — both mechanisms were given directly)
```java
public String loadApiKey(String path) {
    try {
        FileInputStream fis = new FileInputStream(path);   // never closed
        byte[] data = fis.readAllBytes();
        return new String(data).trim();
    } catch (IOException e) {
        e.printStackTrace();
        return null;   // swallowed — API key missing at startup fails silently, later
    }
}
```
`IOException` is checked because it extends `Exception` but not `RuntimeException`. The compiler enforces only this rule. Unchecked exceptions (`NullPointerException`, etc.) extend `RuntimeException` or `Error`. "The computer expects it" is not part of the rule. It is a plain type-hierarchy check.

The second bug is separate: `fis` is never closed. The explanation "the GC cannot collect it, so memory grows" is wrong. The GC does not track or release OS file handles. The real cost is a **leaked file descriptor**, a limited OS resource. If this code runs thousands of times a day, the process reaches its fd limit and throws `IOException: Too many open files`. This is a different failure from `OutOfMemoryError`.
```java
try (FileInputStream fis = new FileInputStream(path)) {
    byte[] data = fis.readAllBytes();
    return new String(data).trim();
} catch (IOException e) {
    throw new IllegalStateException("Could not load API key from " + path, e);   // fail fast at startup
}
```
`try-with-resources` runs `fis.close()` when the block exits, on success or on exception. For startup data such as an API key, propagate the exception so the app fails to start. Do not catch, log, and return `null`. A broken startup config must crash immediately. It must not fail later, where the `null` key is used.

**`List.contains()` is an O(n) linear scan for any list type. For a large membership-check set, use a hash-based structure, not a "faster list".** (2026-09-17, Q3, scored 5/10 — two Socratic nudges were needed; no direct hint text)
```java
List<Long> seenOrderIds = new LinkedList<>();
// ...
if (seenOrderIds.contains(incomingId)) { ... }   // fine at 50 entries, a hot spot at 500,000
```
`LinkedList.contains()` (like `ArrayList.contains()`) walks the list element by element until it finds a match or reaches the end. Each call is O(n). A day can accumulate 500,000 IDs, and every incoming order runs the check. One late-day duplicate check can make 500,000 comparisons.
```java
Set<Long> seenOrderIds = new HashSet<>();
if (seenOrderIds.contains(incomingId)) { ... }   // O(1) average, regardless of set size
```
The fix is `HashSet`, not just "a `Set`". A `TreeSet` is also a `Set`, but each lookup is O(log n) because a sorted tree backs it. `HashSet` uses the same hashing and bucket mechanism as `HashMap`. It computes `hashCode()`, jumps to a bucket, and compares only against the other entries in that bucket. It does not traverse the collection. Reflex: for any repeated `.contains()` call in a hot path, ask whether a list (linear scan) or a hash-based structure (near-constant time) backs it. Name the implementation (`HashSet`/`HashMap`), not only the interface.

**Binary search works only on sorted data. Sort once, search many times. Otherwise use a loop or a `HashMap` index.** (2026-10-11, Q3, scored 6/10 — one hint was needed to find the sorted-list precondition)
```java
int find(List<Order> orders, long id) {
    int lo = 0, hi = orders.size() - 1;
    while (lo <= hi) {
        int mid = (lo + hi) / 2;
        long v = orders.get(mid).getId();
        if (v == id) return mid;
        if (v < id) lo = mid + 1;     // discards the left half
        else hi = mid - 1;
    }
    return -1;
}
```
The line `lo = mid + 1` discards the left half. This is safe only if all left values are smaller. If the list is unsorted, the method returns `-1` for IDs that exist. The test passes in dev (sorted test data). It fails in production (a merge of two sources, or an `ORDER BY` on another column).

Cost: binary search is O(log n) per search. Sorting costs O(n log n) once. For one search, an O(n) loop is cheaper. For many searches on the same data, sort once. Then each search is O(log n).

A `HashMap<Long, Order>` is an index, not a result cache. Build it once in O(n). Each lookup by ID is then O(1) on average. Keep it in sync when orders change. Reflex: write the binary-search precondition as a comment or an assertion.

---

## 2. OOP, Design Patterns & Low-Level Design

**Publishing an event is a different decoupling mechanism from injecting an interface. Whether a listener runs sync or async decides if a failure is loud or invisible.** (2026-09-06, Q1, scored 4/10 — the mechanism was corrected twice)
```java
// Before: OrderService knows about every collaborator directly
public void placeOrder(Order order) {
    orderRepository.save(order);
    emailService.sendConfirmation(order);
    inventoryService.decrementStock(order);
    analyticsService.recordPurchase(order);
}
```
Dependency inversion (injecting `EmailService` behind an interface) means `OrderService` still calls a *known* collaborator directly. It knows who it talks to, but not the concrete class. Event publishing is a different shape of decoupling. `OrderService` has **zero knowledge of who listens or how many listeners exist**.
```java
// Publish side — no "topic," no annotation on the publish call itself
@Service
public class OrderService {
    private final ApplicationEventPublisher publisher;
    public void placeOrder(Order order) {
        orderRepository.save(order);
        publisher.publishEvent(new OrderPlacedEvent(order));
    }
}

// Listener side
@Component
public class InventoryEventListener {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        inventoryService.decrementStock(event.getOrder());
    }
}
```
By default, no messaging broker or "topic" is involved. This is the plain in-process Spring `ApplicationEventPublisher`/`@EventListener` mechanism. It is not Kafka/RabbitMQ terminology. State this difference clearly in an answer.

The scenario tests exception handling. **A plain `@EventListener` runs synchronously, on the same thread and in the same transaction as the publisher.** If `InventoryEventListener` throws, the exception propagates through `publishEvent()` into `placeOrder()`. If `placeOrder()` is `@Transactional`, the whole transaction rolls back, including the order row. Candidates often assume the opposite ("the event system swallows the failure"). The dangerous case is the opt-in one:
```java
@EventListener
@Async
public void onOrderPlaced(OrderPlacedEvent event) { ... }
```
`@Async` moves the listener to a separate thread. When it runs, `placeOrder()` has already returned and its transaction has committed. An exception here rolls back nothing and tells the caller nothing. Unless code catches and logs it, it can disappear silently. Reflex: a sync listener gives coupled failure (it can break the publisher's transaction). An async listener gives invisible failure (nothing rolls back, nothing surfaces). Name which one the design uses before you reason about a throw.

---

## 5. REST & API Design

**Endpoint naming (verb vs. noun), `PUT` vs. `PATCH`, and "day-one correctness" vs. "grows-into-a-problem" are three separate questions. Keep them apart in a design answer.** (2026-09-06, Q2, scored 4/10)
```
// What was proposed — verbs baked into the route
POST   addTask         201
GET    getTask         200
PUT    editTask         200
DELETE deleteTask       200
GET    listTask         200
PUT    markComplete     200
```
REST convention puts the **noun** (the resource) in the path. The **HTTP method** carries the verb. A route that already contains a verb (`addTask`, `markComplete`) duplicates what the method does.
```
POST   /tasks             201 Created
GET    /tasks/{id}        200
GET    /tasks             200
PATCH  /tasks/{id}        200   -- partial update, including "mark complete"
DELETE /tasks/{id}        204 No Content   -- idiomatically no body, so no 200
```
`PUT` is a **full replace** contract: the body must hold the entire resource. Marking one field complete is a **partial update**, which is what `PATCH` is for. Reflex: if an endpoint name contains a verb that the HTTP method already expresses, put that verb in the method choice, not in the URL.

**A hard correctness requirement ("user A cannot see user B's tasks") is a different category from "where does this break as usage grows".** The candidate named multi-tenant authorization correctly and unprompted. But the candidate placed it in the wrong bucket by calling it a *scaling* concern. Authorization is needed **on day one**, when there is more than one real user. It is a correctness gap. It does not "break down" under load. The real scaling bottleneck here is the unbounded list endpoint:
```
GET /tasks          -- fine at 20 tasks, breaks at 200,000, and doesn't need "many users" to hit it
GET /tasks?page=0&size=25   -- bounded regardless of how large the table grows
```
Reflex: when a design question asks "where does this break as it grows", answer about *volume* (unbounded list, missing index, N+1). A feature needed from day one (auth, validation) is a different kind of gap, even if both surface in production.

**A slow third-party API takes down your own service through thread exhaustion. The first fix is timeouts, not async.** (2026-10-11, Q1, scored 5/10)
```java
// No timeout: one stuck call holds one server thread, possibly forever
RestTemplate rt = new RestTemplate();
Rate rate = rt.getForObject(url, Rate.class);
```
`RestTemplate` is a synchronous HTTP client. It blocks the calling thread until the response arrives. Tomcat has a limited thread pool (about 200 by default). When all threads wait on the vendor, no thread is left for any request, including endpoints that do not use the vendor.
```java
SimpleClientHttpRequestFactory f = new SimpleClientHttpRequestFactory();
f.setConnectTimeout(2000);   // ms
f.setReadTimeout(4000);      // ms
RestTemplate rt = new RestTemplate(f);
```
Rules for the failure path:
- Retry only on timeouts and 5xx. Do not retry 4xx.
- Use few attempts and exponential backoff.
- Retry a POST only with an idempotency key. Otherwise the user can be charged twice.
- Add a circuit breaker (Resilience4j). It stops calls to the vendor for a short time after many failures.
- Add a bulkhead: a limited thread pool for this vendor.
- Add a fallback: a cached last-known rate with a short TTL, or a clear error.

Async is not the first fix. A slow vendor still builds up pending work. Do not tell the user "checkout success" before the rate is known, because the price can be wrong. Use async only for work the user does not wait for (for example an email). Reflex: for each outbound call, define the timeout, retry, and fallback.

---

## 6. Concurrency & Locking

**`@Version` does not prevent a lost-update race. It turns a silent race into a loud, detected one. Name the exact exception as well as the mechanism.** (2026-09-06, Q3, scored 3/10)
```java
@Entity
class Ticket {
    @Id Long id;
    String status;
    @Version Long version;
}
```
```sql
-- What actually runs on save, under the hood:
UPDATE tickets SET priority = ?, version = version + 1
WHERE id = ? AND version = ?;
-- 0 rows affected -> someone else already bumped the version since this copy was loaded
```
Hibernate treats zero affected rows as a lock failure. It throws `ObjectOptimisticLockingFailureException` (or the plain JPA `OptimisticLockException`). Naming the exact exception type separates "roughly understands the shape" from "knows the API". The same gap appeared in the Q1 event-listener question above.

**Retry vs. surface-to-user is a judgment about whose mental model is stale. It is not only "which mechanism is available".** A silent retry is correct only when reapplying the *same operation* on fresh data is safe, whatever changed underneath. Examples: an idempotent counter decrement, a stock-quantity update. When two humans edit the same record, a silent retry resubmits Agent B's edit based on B's stale view of the ticket. B never learns that Agent A also changed it. The correct response is to show "someone else changed this, please refresh". The conflict is not a technical retry-until-success problem. A *human* needs corrected information before the next action.

**The structural difference `@Version` adds:** a plain `UPDATE` with no version check always succeeds and silently overwrites the other person's change. The race exists, but nothing makes it visible. `@Version` does not stop the race. It guarantees that the system *detects* the race and throws an exception, instead of succeeding silently. Reflex: describe optimistic locking as "makes conflicts loud", not "prevents conflicts".

---

## 7. Testing & Debugging

**Log levels: the threshold mechanism, live level changes without a redeploy, and the security cost of blanket DEBUG.** (2026-09-07, Q3, scored 5/10)
```java
logger.info("processing order {}", orderId);
```
```
Logger configured at: WARN
Does the INFO line above print?  No.
```
A logger configured at `WARN` emits only `WARN` and above. The logger drops an `INFO` call below the threshold before it reaches any appender. This is the same threshold-hierarchy mechanism as the entry "enabling DEBUG doesn't replace INFO/WARN/ERROR" in `summary2.md`. Here it is seen from the opposite side: raising the threshold *suppresses* lower levels, while DEBUG *includes* everything above it.

**Live level changes without a redeploy: the Spring Boot Actuator `/actuator/loggers` endpoint.** During an incident, a redeploy only to set one class to `DEBUG` costs a full deploy cycle for a temporary diagnostic need.
```
POST /actuator/loggers/com.acme.orders.OrderProcessor
Content-Type: application/json
{ "configuredLevel": "DEBUG" }
```
This changes the effective level on the *running* instance at once. It needs no restart and no new artifact. Set the level back the same way when the investigation ends. This is the general purpose of Actuator: inspect and manage a live app (health, metrics, env, loggers) without touching the deploy pipeline. Use it as a first guess for any question about changing something on a running Spring Boot service.

**Blanket DEBUG in production is a data-exposure risk, not only a volume or disk-space risk.** `DEBUG` logging often includes data that `INFO`/`WARN` do not: request bodies, query parameters, internal object state. Enabling it service-wide can leak sensitive data (PII, tokens, payment details) into log storage. This is in addition to the disk and performance cost already noted in the log-level entry of `summary2.md`.

---

## 8. Web Security & Frontend Basics

**Out-of-order fetch responses in a `useEffect` are a race condition. Debouncing narrows the window but does not close it. The real fix is in the cleanup function.** (2026-09-07, Q2, scored 3/10 — the core diagnosis was correct once confirmed; the fix needed two hints and was still not reached)
```jsx
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(data => setResults(data));   // whichever response arrives LAST wins, regardless of which query it answers
  }, [query]);
  return <ResultsList items={results} />;
}
```
Typing "reactjs" fast fires one request per keystroke (`r`, `re`, `rea`, ...). Each request finishes when the network returns it, not necessarily in send order. If the response for `"rea"` arrives *after* the response for `"react"` has rendered, the UI briefly shows the older query's results over the newer ones. The requests did not fail. They **finished** out of order.

Debouncing (delaying the query update until typing pauses) reduces how *often* requests fire. This narrows the window for the race, but does not close it. If the user pauses long enough to fire one request, then resumes typing before that response returns, two requests can be in flight. Nothing guarantees which response lands second.
```jsx
// The actual fix: give each effect run a way to know it's been superseded
useEffect(() => {
  let ignore = false;
  fetch(`/api/search?q=${query}`)
    .then(res => res.json())
    .then(data => { if (!ignore) setResults(data); });
  return () => { ignore = true; };   // cleanup runs for the PREVIOUS render's effect
}, [query]);
```
React runs the cleanup function of the previous render's effect just before it runs the next effect. When `query` changes again, the old effect's `ignore` flag becomes `true`. When its stale response arrives, the `if (!ignore)` check discards it silently. (`AbortController` is the other standard tool for the same purpose: it cancels the previous `fetch` inside the same cleanup.) Reflex: the `useEffect` cleanup function is not only for unmount or unsubscribe. It is the general hook for "the previous render's in-flight work is now obsolete".

**Timing-dependent bugs are usually *harder* to reproduce locally, not easier. Low, consistent localhost latency hides the race.** The instinct "the logic is deterministic, so it will reproduce locally" is backwards for a race like the one above. On localhost, requests almost always finish in send order, because network latency is near zero and consistent. The race rarely triggers. Production network conditions (load, geography, jitter) vary. They let responses finish out of order. For this reason such bugs can pass all local tests and appear only for real users. This generalizes: suspect that any bug triggered by *timing variance* is harder to catch on a fast, consistent dev machine.

**The browser enforces CORS, not the server. The fix is a response header from the backend.** (2026-10-11, Q2, scored 5/10)
```
Browser console:
Access to fetch at 'http://localhost:8080/api/orders' from origin 'http://localhost:3000'
has been blocked by CORS policy.
```
Postman is not a browser, so it has no CORS check. The server still receives the request. The browser only refuses to let your JavaScript read the response. React at `:3000` stays a different origin from Spring at `:8080`. The backend does not make it "same origin". It allows the origin with a header:
```
Access-Control-Allow-Origin: http://localhost:3000
```
```java
@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration c = new CorsConfiguration();
    c.setAllowedOrigins(List.of("http://localhost:3000"));
    c.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    c.setAllowedHeaders(List.of("Content-Type", "Authorization"));
    UrlBasedCorsConfigurationSource s = new UrlBasedCorsConfigurationSource();
    s.registerCorsConfiguration("/api/**", c);
    return s;
}
```
For `PUT`, `DELETE`, or JSON requests, the browser first sends an `OPTIONS` preflight request. The backend must answer it with the correct `Access-Control-*` headers.

The risk of `*` is not DDoS. CORS does not stop DDoS. With `*`, any website can make a user's browser call your API and read the response. This is dangerous when the API uses cookies or tokens. Allow only known origins. Reflex: open the browser Network tab, and read the `OPTIONS` request and the `Access-Control-*` headers.

---

## 9. System Design & Architecture Basics

**An unbounded in-process cache is a memory-leak bug. It differs from the staleness problem of caching. The real trade-off of adding TTL is a staleness window, not CPU overhead.** (2026-09-07, Q1, scored 6/10 — a hint was needed to name the fix, and one misconception about the trade-off persisted)
```java
private static final Map<Long, UserProfile> cache = new HashMap<>();

public UserProfile getProfile(Long userId) {
    if (cache.containsKey(userId)) return cache.get(userId);
    UserProfile profile = userRepository.findProfile(userId);
    cache.put(userId, profile);
    return profile;
}
```
`summary.md` and `summary2.md` already cover the *staleness* failure of caching (a too-long TTL serves stale data). This is a different failure. Nothing here expires or evicts entries, so every distinct `userId` ever looked up stays in the map for the life of the process. Over weeks in production the map only grows. Eventually the heap fills and the service throws `OutOfMemoryError`. A plain `HashMap` has no built-in time or size limit. The limit must come from elsewhere: a different data structure (`LinkedHashMap` with `removeEldestEntry` for LRU, or a library such as Caffeine, both in-process), or an external store such as Redis with a TTL.

**The trade-off of TTL-based expiry is a staleness window, not a performance cost.** A timestamp comparison to check expiry is almost free. It is not a meaningful source of request latency. The real cost: for up to the TTL duration, the cache can serve an entry after the underlying row changed. A shorter TTL reduces staleness but increases DB hits. A longer TTL does the reverse. Name this tension (staleness vs. DB load) as the trade-off, not "checking expiry takes time".

**Scan an entity's own fields for a concrete leak before you reach for a general mechanism such as N+1. Know the exact name of the exception that fires.** (2026-09-13, Q3, scored 4/10 — the candidate proposed the DTO + JOIN FETCH fix unprompted, but never named the entity-specific leak and mislabeled the exception)
```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();   // returns the entity directly
}

@Entity
public class User {
    private Long id;
    private String email;
    private String passwordHash;                          // leaked to every client, verbatim
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
}
```
The sharpest, entity-specific problem is that `GET /users/{id}` leaks `passwordHash` to any client. It is easy to miss when you reach straight for general "do not expose entities" mechanisms. The second problem is one **`LazyInitializationException`**, not "N+1". The `findById` transaction closes as soon as the repository call returns. Jackson later tries to serialize the still-lazy `orders` collection outside any open session. N+1 is a different bug. It happens when code loops over *many* parent entities and each one triggers its own lazy-load query. A single `findById` call has no loop to multiply.
```java
public record UserResponse(Long id, String email, int orderCount) {}   // passwordHash never mapped out at all

@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);
```
Adding a computed `orderCount` field six months later is now simple and needs no schema change. Map `orders.size()` (or a dedicated `COUNT` query) onto the DTO. The DTO is a view shape that you fully control. Adding the same computed field to the `@Entity` would mix persistence mapping with response shaping. It would also hit the same lazy-loading and closed-session problem from the first bug.

**`fetch()` rejects its promise only on a network-level failure. A real HTTP error status still counts as a "successful" fetch. This is the opposite failure mode from a CORS block.** (2026-09-13, Q2, scored 3/10 — first misdiagnosed as a backend design issue; the `res.json()` mechanism and the fix were given directly)
```jsx
useEffect(() => {
  fetch(`/api/orders/${orderId}`)
    .then(res => res.json())          // parses successfully even for a 404 body
    .then(data => setOrder(data))     // sets order to { error: "Order not found" }
    .catch(() => setError('...'));    // never fires — nothing rejected
}, [orderId]);
```
To `fetch()`, a `404` with a valid JSON error body is a completed request. `res.json()` parses whatever came back, regardless of `res.status` or `res.ok`. The `.then()` chain therefore continues into `setOrder` with the error payload and never reaches `.catch()`. The fix is to `throw` inside the first `.then()` when `!res.ok`. This rejects the chain and routes it to `.catch()`:
```jsx
.then(res => { if (!res.ok) throw new Error(`fetch failed: ${res.status}`); return res.json(); })
```
This is the mirror image of a CORS failure. CORS rejects the `fetch()` promise at network level with a `TypeError`, before any `.then()` runs. There is no readable `Response` to inspect. A real `404` returns a full, parseable body. Reflex: `.catch()` fires only for network or CORS-level failures, never for the server's HTTP status code. Checking `res.ok` is the code's job.

**Cache invalidation on write, applied to a mutable field such as a profile picture:** when the underlying data changes, evict the specific key. The next read is then a guaranteed miss and repopulates from the source of truth. This is the same invalidate-on-write pattern as the pricing-cache entry in `summary2.md`, applied to a different field.
```java
public void updateProfilePicture(Long userId, byte[] newPicture) {
    userRepository.updateProfilePicture(userId, newPicture);
    cache.remove(userId);   // next getProfile() call is a guaranteed miss, re-fetches from DB, re-caches fresh
}
```

**Vertical scaling is a "bigger box". Horizontal scaling is "more boxes". A horizontal setup adds two separate things: the load balancer's routing decision and the statelessness requirement.** (2026-09-17, Q1, scored 5/10 — the vertical/horizontal split was correct unprompted; both follow-on parts had gaps)
```
Vertical:    2 vCPU/4GB  ->  8 vCPU/16GB          (same one instance, more powerful)
Horizontal:  1 instance  ->  3 instances + LB     (same-sized instances, more of them)
```
With multiple instances, a load balancer answers two *different* questions. A health check covers only one. (1) **Is this instance eligible?** Health checks remove a failing instance from rotation. (2) **Which eligible instance gets this request?** A separate routing algorithm decides. Examples: round-robin (cycle through instances in order) and least-connections (send to the instance with the fewest in-flight requests). This session's answer named only the health-check side. It missed the per-request routing mechanism.

**Horizontal scaling requires the app to be stateless. It does not require networking or port configuration.** The load balancer can send any request to any instance. So nothing request-specific can live only in one instance's process memory: a session map, an in-memory cache, a counter. If it does, request #2 from the same user can reach a different instance than request #1. That instance has none of the state the first one built. Fix: move that state to something all instances share (Redis, the database). This is the same constraint as the "stateless services" entry in `summary.md` (an in-process session `Map` breaking behind round-robin), asked here from the scaling-trade-off angle instead of the session-storage angle.

---

## 10. Tooling: Git, Build & Environment

**A green, clean-looking PR still needs a reviewer to check what CI cannot check. "Does it relate to one thing" (cohesion) is only one of several angles.** (2026-09-17, Q2, scored 5/10 — cohesion was named unprompted; a hint was needed to reach untested logic; the candidate stopped there and did not name the other angles)
```
CI checks:             Does it compile? Do the tests that already exist pass?
Still open questions:  Is there logic in this diff with no test covering it at all?
                        Does it actually implement what the ticket asked for?
                        Is there a security gap no test was written to catch?
                        Does it fit the codebase's architecture (right layer, no duplicated logic)?
```
This is the same reflex as the entry "green CI proves the tests that exist pass, not that they're the right tests" in `summary2.md`. Here it becomes a concrete reviewer checklist for one ordinary 40-line diff:
- **Untested logic:** an edge case, an error path, or a null/empty input that no test exercises. This session's answer reached it after a hint.
- **Requirement mismatch:** clean, fully tested code that solves the wrong problem still passes CI.
- **Security gaps:** a new endpoint without an auth check, a hardcoded secret, unvalidated input. No test fails, because nobody wrote one.
- **Architectural fit:** business logic in a controller instead of a service, or a duplicate of an existing utility. Both compile and pass tests.

Reflex: before you approve any green PR, ask "if this were subtly wrong, would CI have caught it?" If not, put review attention there.
