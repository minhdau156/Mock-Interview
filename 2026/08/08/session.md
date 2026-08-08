# Mock Interview — Junior Backend Developer
**Date:** 2026-08-08

**Categories covered (3-question session, seed-random start (seed=1344, start=5, step=7), categories 2/3 skipped because they repeat 2026-08-07's topics — OOP/LLD, Databases — per explicit request; sequence otherwise: REST → OOP → System Design → Concurrency → Databases):** REST & API Design, System Design & Architecture Basics, Concurrency & Locking

---

## Q1 — REST & API Design

**Question:** Your service calls a third-party shipping-rate API from a controller endpoint (`GET /orders/{id}/shipping-rate`) to show the customer a live quote. In a code review, a teammate flags your integration code:

```java
public ShippingRate getRate(Order order) {
    return shippingApiClient.getRate(order.getDestination(), order.getWeight());
}
```

No timeout is configured on the HTTP client, and there's no try/catch around the call. What's your concern with this as-is, and what would you change?

**Candidate asked for the scenario to be spelled out in more detail before answering** (not a hint on the answer itself — just a restatement of the scenario).

**Candidate's answer:**
> "i think when it don't have the timeout configured it will slow and crash the app so the server will down / no try catch exception so if this throw exception my app will pause and log the error it don't run / so when i try i will add the api gateway to check the timeout for each request so if the request it take 30s or more it will notice the user / and i will add the try catch so my app just log the error don't pause the app and it keep running"

**Feedback given:**
- Strong: correctly flagged both the missing timeout and missing try/catch as real problems, and connected them to "something breaks for the whole app" — the right instinct.
- Missing: no timeout doesn't "crash" the server — it causes **thread pool exhaustion** (threads pile up waiting indefinitely, and once the pool is exhausted no thread is left to serve any request, including unrelated ones).
- Missing: an uncaught exception in a Spring controller doesn't "pause the app" either — Spring's default handling catches it and turns it into a generic 500 for *that one request*. The real problem is that 500 is ugly/unhelpful instead of a friendly fallback message.
- Missing: "API gateway" mislabels the fix — the actual fix is configuring **connect/read timeouts directly on the HTTP client** (e.g. `RestTemplate`/`WebClient` config), no new infrastructure needed.
- Missing: the catch block should return a graceful fallback (default rate / "unavailable" message) instead of just logging; a simple retry-with-backoff is also standard.

**Score: 5/10**

---

## Q2 — System Design & Architecture Basics

**Question:** Your team's app runs on a single server. Response times have been creeping up under load, and someone suggests "let's just get a bigger server" while someone else says "let's run two servers behind a load balancer instead." Walk me through: what's the actual difference between those two approaches, and what has to be true about your app for the second one ("two servers") to actually work correctly?

**Candidate's answer:**
> "the different is the bigger server can handle request more faster... but two server it not like that it will route the request to both server so the first server don't need to have the pressure / i think the first one is vertical scale and the second is horizontal scale / i the second approach if i want to use it and make it work properly i need to my app is completely stateless... if we use the session (stateful) it can save the information of the user in the first server and the second don't know about it so when the request goes to the second it will get unauthorized error / so we need to use JWT token for this and this the easiest way"

**Feedback given:**
- Strong: correctly and unprompted named vertical vs. horizontal scaling; correctly diagnosed why horizontal scaling needs statelessness (session on one instance is invisible to the other → unauthorized on the "wrong" server); proposed JWT unprompted as a valid fix.
- Missing: the comparison should also cover redundancy — a single bigger server has a hard ceiling and is a single point of failure; horizontal scaling adds fault tolerance, not just capacity.
- Missing: JWT is one valid fix; a shared server-side session store (Redis) is the sibling standard answer, and sticky sessions (load balancer pins a client to one server via cookie/IP hash) is a simpler-but-fragile bolt-on alternative.
- Missing: the general principle — anything kept only in one instance's memory (cache, rate limiter, WebSocket connection) breaks the same way past one instance, not just sessions.

**Score: 7/10**

---

## Q3 — Concurrency & Locking

**Question:** A teammate adds a "send confirmation email" feature to the order-placement endpoint:
```java
public void placeOrder(Order order) {
    orderRepository.save(order);
    new Thread(() -> emailService.sendConfirmation(order)).start();
}
```
Works fine locally and in staging. In production under real traffic, the app throws `OutOfMemoryError` and becomes unresponsive. What's going wrong, and what would you change?

**Candidate asked for clarification** on why `new Thread` was used at all — given a brief explanation of the fire-and-forget intent before answering.

**Candidate's answer:**
> "i think when get the thread it will cost the memory, the thread for the current request and the thread for send confirmation so if have 5000 request in few minute it can have 10000 thread so if each thread can not handle it, it also create and the memory have full space and having out of memory error so / in this one i can use MessegeQueue for the send confirmation by email so that we just need create the MQ and push to it we don't need to create the new thread"

**Feedback given:**
- Strong: correctly identified the core mechanism with only a light nudge — unbounded thread creation under load costs memory and has no cap, leading to OOM. Proposed a legitimate fix (message queue).
- Missing: precise mechanism — each `Thread` reserves ~512KB–1MB of stack just to exist; 10,000 threads is already gigabytes of stack alone.
- Missing: the compounding effect — if the email service is slow, threads pile up faster than they finish, so concurrently-alive count keeps climbing under sustained load.
- Missing: the more direct, surgical fix for *this* bug is a **bounded thread pool** (`ExecutorService`/`Executors.newFixedThreadPool(n)`) — recycles a capped number of threads and queues excess work. A message queue is a good production answer too but is a bigger architectural change (new infrastructure); the thread pool is the smaller fix to the actual bug shown.
- Missing: naming `OutOfMemoryError: unable to create new native thread` as the specific, distinct OOM flavor here (an OS thread-limit issue, not a heap-filling-with-objects issue).

**Score: 7/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | REST API — timeouts/error handling when consuming a third-party API | 5/10 |
| 2 | System Design — vertical vs. horizontal scaling and statelessness | 7/10 |
| 3 | Concurrency — raw `Thread` creation vs. thread pools | 7/10 |

**Overall: 6.3/10**

**Strengths:**
- Correctly named both scaling strategies (vertical/horizontal) and the statelessness requirement for horizontal scaling unprompted, with a working fix (JWT).
- Correctly diagnosed the root cause of unbounded thread creation → memory exhaustion with only a light nudge, and proposed a legitimate architectural fix (message queue).
- Good instinct throughout: reasoning about resource costs (memory, threads) rather than reciting definitions.

**Areas to review before the real interview:**
- Third-party API integration precision (Q1) — thread-pool exhaustion vs. "crash," Spring's default 500 handling vs. "app pauses," client-side timeout config vs. "API gateway," and adding a graceful fallback response.
- Vertical vs. horizontal scaling completeness (Q2) — redundancy/single-point-of-failure angle, and sticky sessions as the simpler alternative fix.
- Thread pools as the direct fix for unbounded thread creation (Q3) — `ExecutorService`/fixed thread pool is the first-reach-for fix, ahead of the bigger message-queue redesign.
