# Mock Interview — Junior Backend Developer
**Date:** 2026-07-06

**Categories picked this session:** Tooling: Git, Build & Environment → System Design & Architecture Basics → Web Security & Frontend Basics → Testing & Debugging → Concurrency & Locking
(seed = 1039, starting_index = 10, step = 9)

---

## Question 1 (Tooling: Git, Build & Environment)

**Q:** You're reviewing a teammate's pull request. It's titled "fixes stuff" and contains one giant commit: a bug fix for a null pointer issue, a rename of a service class, and a reformatting of an unrelated config file — all mixed together. What's your feedback to them, and why does it matter beyond just "it looks messy"?

**A:** i think that should commit each problem instead because it can make the commit mixed together and hard to review. i think they should use 3 commit like fix: fix the null pointer issue in ...., chore: rename the ... service, refactor: reformat the ... config file. it matter than because you will give them the idea and they can change it later and they will rarely make the commit like this again

**Score: 7/10**

**Strong:**
- Correctly identified the core fix: split into separate, logical commits instead of one bundled commit.
- Used sensible conventional-commit prefixes (`fix:`, `chore:`, `refactor:`), showing awareness of a real-world commit convention.
- Recognized that mixing changes together makes the PR harder to review.

**Missing:**
- The "why it matters" reasoning leaned on coaching the teammate's future behavior rather than the technical consequences. Two concrete ones worth naming: revert safety — if the null pointer fix turns out wrong and needs reverting, you'd also revert the unrelated rename and reformat; and blame/bisect clarity — six months from now, `git blame` on the renamed class or the config file will point to a commit titled "fixes stuff" that also happens to contain a bug fix.
- Didn't mention that reviewers can't approve/reject pieces independently — a single commit forces an all-or-nothing review.

**Tip:**
- When judging a commit boundary, ask "if I had to revert or `git blame` just this one change in 3 months, could I isolate it cleanly?"

---

## Question 2 (System Design & Architecture Basics)

**Q:** Your team's product-details endpoint (`GET /products/{id}`) is getting hammered — most of the traffic is the same handful of popular products being fetched over and over, and the database is starting to show it under load. A teammate suggests adding a cache in front of the DB call. What would you actually cache, how would you decide how long to keep it, and what could go wrong if you get that duration wrong?

**A:** it will cache the data that are get from the database, for example, the client make a query and if multiple request subsequently make this request it will down the DB, so when have the data in cache we just query for the first time, after that the data is from the cache not the DB, it will protect the DB. i think the endpoint get /products/{id} it will usually have this request so i think i will set the TTL about 1 - 2 minutes. if we sett the wrong time like we set it longer like 10 minutes it will make the application appear the data that is not match in the DB and it can make some bug like the count of product in DB is 1 but have 2 - 3 request to buy it if we don't handle the case that the quantity is < 0 it will the bug in our system. also that if we set the time to fast, the DB is still down, example when the iphone is launch the people will go to the website a lot for buying and if we set the TTL about 10 - 30 seconds it still down the website

**Score: 8/10**

**Strong:**
- Correct mental model of cache-aside: first request hits the DB and populates the cache, subsequent requests read from cache.
- Genuinely good concrete example of the stale-data trade-off with a long TTL — cached inventory count going stale while real stock drops, leading to overselling.
- Correctly reasoned that too-short a TTL doesn't fully solve the problem under bursty traffic.

**Missing:**
- The short-TTL failure mode has a name: cache stampede / thundering herd — many requests miss the cache at the exact same moment and all hit the DB simultaneously.
- TTL isn't the only tool for staleness — invalidating the cache on write (updating/evicting the entry the moment stock changes) is the standard complement.

**Tip:**
- Choose TTL per field based on how fast it changes and the cost of staleness — long TTL for description/images, short-or-invalidate-on-write for stock/price.

---

## Question 3 (Web Security & Frontend Basics)

**Q:** You're building a small React page that calls your own backend API to load a user's profile. In dev, your frontend runs on localhost:3000 and your backend on localhost:8080. When you call the API with fetch, the browser console shows a CORS error and the request never even seems to reach your controller's logic (no log line printed). What's actually going on here, and what needs to change to fix it?

**A:** it is the policy in the browser because the backend server is not permit the diffrent application that it don't know get into the api because it scared that it will be get the resoured without permission and to protect the backend server. so if we want to fix just have some configuration in backend allowing the port localhost:3000 can use the api to fetch the data

**Score: 6/10**

**Strong:**
- Correctly identified this as a browser-enforced policy issue, not a bug in the fetch code.
- Landed on the right fix direction: the backend needs to explicitly allow the frontend's origin.

**Missing:**
- The mechanism is backwards: the browser is what blocks it, based on response headers (`Access-Control-Allow-Origin`, etc.) the backend must send.
- The clue "no log line printed" points to a preflight `OPTIONS` request — if the backend doesn't answer that correctly, the browser never sends the real request at all.

**Tip:**
- Check the Network tab first — a failing `OPTIONS` request right before the real request means it's a preflight issue; fix is CORS config, not controller logic.

---

## Question 4 (Testing & Debugging)

**Q:** You're debugging an issue where a scheduled job occasionally skips processing some orders — but it only happens in production, roughly once every few days, and you can't reproduce it locally. A teammate suggests just adding a bunch of System.out.println statements and redeploying to prod. Is that reasonable, and how would you actually track this down?

**A:** (after one hint about gathering evidence without redeploying, and a follow-up prompt to elaborate) i think i will log when processing the order and have the the span on the processing function so it can easily to track, i think the System.out.println is good

**Score: 4/10** *(needed two hints to get here)*

**Strong:**
- Mentioned logging around order processing and adding a trace/span to correlate execution.

**Missing:**
- Saying `System.out.println` is "good" contradicts the point of the hints — no log levels, not searchable/aggregated, requires a redeploy per adjustment.
- No hypothesis for why a scheduled job would occasionally skip orders — e.g. the job running on two instances/threads at once (race condition), or an exception on one order being silently swallowed inside the loop.
- No mention of what context to log (order id, timestamp, instance/thread id) to narrow down the pattern.

**Tip:**
- For intermittent bugs, logs that are always on and cheap beat println + redeploy + hope.

---

## Question 5 (Concurrency & Locking)

**Q:** A simple in-memory `int count` field in a Spring service, incremented on every request via `count++`. Under load testing, the final count is consistently lower than the actual number of requests. What's going wrong, and how would you fix it?

**A:** it occurr because it have the race condition, when 2 thread using same resource it is count variable... [detailed walkthrough of read-increment-write interleaving between thread A and thread B]... so if we want to fix it we need to use some lock, the easily approach is use MUTEX (mutual exclusive); it will lock the thread until this lock is done is will release and the next thread cotinues proccessing

**Score: 8/10**

**Strong:**
- Correctly explained why it's a race condition: `count++` is actually read → increment → write, with a clear walkthrough of the lost-update interleaving.
- Correctly identified the fix direction: serialize access with a lock.

**Missing:**
- Named the general OS concept (mutex) but not the Java-specific tool: `synchronized` on the method, or better, `AtomicInteger.incrementAndGet()` (lock-free CAS, cheaper under load).
- Didn't mention this `StatsService` is a Spring singleton bean, so `count` is shared mutable state across every concurrent request thread.

**Tip:**
- For simple numeric counters, reach for `AtomicInteger`/`AtomicLong` before `synchronized`.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Commit hygiene (Tooling: Git, Build & Environment) | 7/10 |
| 2 | Caching & TTL trade-offs (System Design & Architecture) | 8/10 |
| 3 | CORS mechanics (Web Security & Frontend Basics) | 6/10 |
| 4 | Debugging an intermittent production bug (Testing & Debugging) | 4/10 |
| 5 | Race condition in a shared counter (Concurrency & Locking) | 8/10 |

**Overall: 6.6/10**

**Strengths:**
- Strong at walking through concrete mechanics once committed to an explanation — the thread-interleaving walkthrough on Q5 and the inventory-overselling example on Q2 were genuinely solid, junior-appropriate reasoning.
- Good instinct for the right category of fix (split commits, add a cache, add a lock) even when details underneath were incomplete.

**Areas to review before the real interview:**
- CORS mechanics: the browser enforces the policy based on headers the backend returns, not the backend "refusing" the request — and preflight `OPTIONS` requests explain why a blocked call sometimes never reaches the controller at all.
- Debugging under pressure without hints: form a hypothesis about the mechanism (race condition, silently swallowed exception) rather than jumping straight to a tooling answer.
- Naming language-specific tools: consistently understands concepts (mutex, cache) but stops short of naming the Java/Spring-specific idiom (`AtomicInteger`, `synchronized`, cache invalidation on write).
- Tying "why it matters" to technical consequences: ground answers in things like revert safety and git blame/bisect clarity, not just behavioral coaching.
