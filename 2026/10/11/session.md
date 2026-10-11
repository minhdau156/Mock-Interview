# Mock Interview — Junior Backend Developer
**Date:** 2026-10-11

**Seed:** 21:15:34 → 76534. Start = 5, step = 3. Sequence: REST & API Design → Web Security & Frontend Basics → Core Java & CS Fundamentals.

---

## Q1 — REST & API Design (consuming third-party APIs)

**Question:** A Spring Boot checkout calls a slow third-party exchange-rate API with `RestTemplate`. Some calls hang for minutes. The service stops responding. Why? What do you change? What do you do on failure?

**Candidate's answer:** `RestTemplate` is synchronous, so a slow vendor makes the API slow. Use async. Show "checkout success" and process in the background. On failure, retry and tell the customer to retry.

**Score: 5/10**

**Strong:**
- Found that `RestTemplate` is synchronous.
- Thought about retry and user messages.

**Missing:**
- Thread exhaustion. Each slow call holds one server thread. The pool is limited (about 200 in Tomcat). When all threads wait, the server cannot accept new requests.
- Timeouts. Set a connect timeout and a read timeout.
- Async is not the first fix. Do not tell the user "checkout success" before the rate is known.
- Retry rules. Retry only on timeouts and 5xx. Use few attempts and backoff. Retry a POST only with an idempotency key.
- Circuit breaker and fallback (cached rate or clear error).

**Tip:**
- For each outbound call, define the timeout, retry, and fallback.

---

## Q2 — Web Security & Frontend Basics (CORS)

**Question:** React at `localhost:3000` calls Spring at `localhost:8080`. The browser blocks the call with a CORS error. Postman works. Why? What is the fix? Is `*` OK in production?

**Candidate's answer:** CORS blocks different origins. The backend must accept the React origin. Allow-all is dangerous: easy to call the API, DDoS, and server shutdown.

**Score: 5/10**

**Strong:**
- Knows the browser blocks cross-origin calls.
- Knows the backend must allow the React origin.
- Knows `*` is risky.

**Missing:**
- The browser enforces CORS. Postman is not a browser. The server still receives the request.
- "Same origin" is wrong. React stays a different origin. The backend sends `Access-Control-Allow-Origin`.
- Spring fix: `@CrossOrigin`, `WebMvcConfigurer`, or `CorsConfigurationSource`. Preflight `OPTIONS` must work.
- The `*` risk is not DDoS. Any website can make a user's browser call the API and read the response.

**Tip:**
- Read the `OPTIONS` request and the `Access-Control-*` headers in the browser Network tab.

---

## Q3 — Core Java & CS Fundamentals (binary search)

**Question:** A `find(List<Order>, id)` method uses binary search. In production it sometimes returns `-1` for IDs that exist. What must be true about the list? What is the Big-O? When use a `HashMap`?

**Candidate's answer (after one hint):** The list must be sorted ascending by ID. The method is O(log n), but sorting costs O(n log n), so a loop at O(n) is better here. Use a `HashMap` to cache results when there are many searches.

**Score: 6/10** (one hint needed)

**Strong:**
- Found the sorted-list precondition.
- Found that sorting cost can remove the benefit of binary search for one search.
- Found that many searches favor a map.

**Missing:**
- Without a hint, the sorted-list precondition was not found.
- Sorted once, then many searches: O(n log n) once, then O(log n) per search. This is the case where binary search wins.
- A `HashMap<Long, Order>` is not a result cache. It is an index built once in O(n). Each lookup is then O(1) on average. Use it for many lookups by ID. Keep it in sync when orders change.
- Production data can come from a merge of two sources or from an `ORDER BY` on another column. The list may be unsorted without notice.

**Tip:**
- Before binary search, write the precondition as a comment or an assertion.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | REST & API Design — third-party API timeouts | 5/10 |
| 2 | Web Security & Frontend Basics — CORS | 5/10 |
| 3 | Core Java & CS Fundamentals — binary search preconditions | 6/10 |

**Overall: 5.3/10**

**Strengths:**
- Good link between cause and effect after a hint.
- Thinks about cost trade-offs (sort cost vs loop).

**Areas to review before the real interview:**
- Timeouts, thread pool exhaustion, retry rules, and circuit breakers.
- Who enforces CORS (the browser), preflight `OPTIONS`, and the real risk of `*`.
- Precondition of binary search and the `HashMap` as an index.
