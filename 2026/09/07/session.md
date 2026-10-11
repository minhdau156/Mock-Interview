# Mock Interview — Junior Backend Developer
**Date:** 2026-09-07

**Categories covered (3-question session, date-based seed (no timestamp available): seed = 7×137 + 9×31 = 1238, start=9, step=9; sequence: System Design & Architecture Basics → Web Security & Frontend Basics → Testing & Debugging; opening category (9) differs from 09-06's opener (2) per the no-repeat-opener rule; all three sub-topics were checked against every prior session back to 07-04 to avoid reusing a scenario/mechanism already asked — System Design landed on the untested cache-eviction/unbounded-growth angle of "caching basics" (the TTL-staleness angle had already been covered on 07-06 and 08-12), Web Security & Frontend used the untested out-of-order-fetch-response race angle of "calling a backend API from React" (the CORS-preflight angle had already been covered three times: 07-06, 07-10, 07-14), and Testing & Debugging used log levels as the direct subject for the first time — previously only flagged as a missing concept (07-05) but never actually tested):** System Design & Architecture Basics, Web Security & Frontend Basics, Testing & Debugging

---

## Q1 — System Design & Architecture Basics

**Question:** You're reviewing a teammate's PR. They've added a simple in-process cache to speed up a "get user profile" endpoint:

```java
private static final Map<Long, UserProfile> cache = new HashMap<>();

public UserProfile getProfile(Long userId) {
    if (cache.containsKey(userId)) {
        return cache.get(userId);
    }
    UserProfile profile = userRepository.findProfile(userId);
    cache.put(userId, profile);
    return profile;
}
```

It works fine in the PR's testing. You leave a review comment.

- What's the actual problem with this cache as the service runs for weeks in production — what happens to it over time, and why does that eventually hurt the app rather than help it?
- Name one concrete strategy to fix this (you don't need exact API syntax — describe the mechanism), and explain what it trades off in return.
- Separately: if a user updates their profile picture, and the old one is still sitting in this cache, how would you make sure they don't keep seeing the stale version?

**Candidate's answer** (needed a hint to move from "the map grows forever" to "it needs an eviction/expiry mechanism," and asked a clarifying question about whether plain `HashMap` supports TTL):
> "i think the problem cache can have is it will make the system have memory leak i will add more the data user profile to the cache and i don;t have the approach to remove it out so when the Map more bigger and it will have the OOM exception in my personal project i use the redis database to cache and it support me to have the TTL of each key without it i can make the db of hashmap more bigger and cause OOM so i think i will check an [cut off]"
>
> (continued after prompt) "when have TTL it can lost the time for checking does it expired so it can make the request have a litte bit slow so when user update the profile picture we should have some approach to invalidate the data in cache for consistency so i will delete the data have the key that is updated and update to DB and query from the DB and add to cache again"

**Feedback given:**
- Strong: correctly diagnosed the root problem — nothing ever leaves the map, so it grows without bound and eventually causes an OOM.
- Strong: named a real, working strategy (external cache with TTL, e.g. Redis) tied to actual personal-project experience.
- Strong: Part 3 nailed — invalidate the stale key on update and let the next read re-populate the cache from the DB.
- Missing/misconception: the trade-off of TTL isn't "checking expiry makes requests a little slower" (a timestamp comparison is essentially free) — the real cost is a **staleness window**: for up to the TTL duration, a user can be served outdated data even after the underlying row changed. Shorter TTL reduces staleness but increases DB load; longer TTL is the reverse.
- Missing: needed a hint to identify that an eviction/expiry mechanism was the fix at all.
- Bonus (not required): in-process alternatives to Redis exist too (`LinkedHashMap` with `removeEldestEntry` for LRU, or Caffeine) — Redis works but adds infrastructure rather than fixing the data structure itself.

**Score: 6/10**

---

## Q2 — Web Security & Frontend Basics

**Question:** A teammate built a search-as-you-type box in React:

```jsx
function SearchResults({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(data => setResults(data));
  }, [query]);

  return <ResultsList items={results} />;
}
```

QA reports: "When I type fast, the results sometimes flash to match an *earlier* letter I typed, even though I've kept typing past it." For example, typing "reactjs" quickly, the box briefly shows results for "rea" *after* it had already shown results for "react".

- Why does this happen — what's actually going on with the network requests as `query` changes rapidly? (This isn't a CORS problem — the requests all succeed.)
- What would you change in this component to fix it — describe the mechanism, not just "add a check somewhere."
- Is this a bug that would show up in your local dev testing, or is it the kind of thing that mostly surfaces in the real world? Why?

**Candidate's answer** (asked a clarifying question about the failure sequence before answering; needed two hints on part 2 and still could not produce the ignore-flag/`AbortController` mechanism):
> "i think it is the problem in the latency in each response ... so when the response of react come first but then the response of re come later it will override the result like you say so in my opinion i think we need to use setTimeOut ... we should use setTimeOut in the query state like setQuery in 2 second so that after 2 second the query will change and then the request will make with that so that we can see the exactly response with what we type"
>
> (Part 3) "i think it will show in the local dev testing because it the logic that can reproduce"
>
> (final hint on ignore-flag/cleanup mechanism) "i don't know"

**Feedback given:**
- Strong: once confirmed, correctly restated the race mechanism — each keystroke fires its own request, and network timing variance lets a later request's response arrive before an earlier one.
- Strong: reached for debouncing as a relevant technique, even though it doesn't fully solve this bug.
- Missing/misconception: debouncing only reduces *how often* requests fire — it narrows the race window, it doesn't close it. The correct fix: track a local flag inside the effect (`let ignore = false`) and return a cleanup function that sets `ignore = true`; React runs the previous render's cleanup before the next effect runs, so a late response checks `if (!ignore)` before calling `setResults` and is silently discarded. (`AbortController`, cancelling the previous fetch in the same cleanup function, is the other standard tool.)
- Missing/misconception: Part 3 was answered backwards — this bug is *harder* to reproduce locally, not easier. Localhost latency is tiny and consistent, so responses almost always finish in the order they were sent; production's variable network conditions are what actually let responses finish out of order, which is why timing-dependent bugs like this often pass local testing and only surface for real users.

**Score: 3/10**

---

## Q3 — Testing & Debugging

**Question:** Your team's order-processing service has an intermittent bug — support says it happens maybe once a day, only in production, and nobody can reproduce it locally. A teammate suggests: "Let's just bump the whole service's logging to DEBUG so we catch it next time it happens."

- What's the actual risk of turning DEBUG logging on for an entire production service, rather than just accepting the extra noise?
- Assume you can't redeploy right now (it's mid-incident). Is there a way to raise the log level for just the one suspect class, live, without a new deployment? What would that involve?
- If a logger is configured at `WARN` level, and the code has a `logger.info("processing order {}", orderId)` call in it — does that line print anything? Walk me through how level filtering actually decides what gets written.

**Candidate's answer** (needed a hint pointing at Spring Boot Actuator for part 2, and never produced an answer for it):
> "it can make the terminal have a lot of log and it can waste disk space and slow down the system i don't know"
>
> (Part 3, after a repeated prompt) "in the part if we set the log to warn we can not see the log at the info level because it is the mechanism of logging the level of warn is higher than info so that we just see the log at WARN level or higher"

**Feedback given:**
- Strong: correctly named the core costs of blanket DEBUG logging — volume, disk space, performance overhead.
- Strong: nailed the level-filtering mechanism unaided — a `WARN`-configured logger only lets `WARN`-and-above through, so an `INFO` call underneath it is silently dropped.
- Missing: no knowledge of `/actuator/loggers` — Spring Boot Actuator's endpoint for viewing/changing a logger's level on a live running instance without a redeploy — even after a direct hint.
- Missing: missed the security/privacy angle of blanket DEBUG logging (it often logs request bodies, query params, or internal state that INFO/WARN wouldn't) even after being told once.

**Score: 5/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | System Design & Architecture Basics (unbounded in-process cache / eviction) | 6/10 |
| 2 | Web Security & Frontend Basics (React out-of-order fetch responses) | 3/10 |
| 3 | Testing & Debugging (log levels & live level changes) | 5/10 |

**Overall: 4.7/10**

**Strengths:**
- Consistently lands on the right root cause once the scenario is concrete — unbounded growth, timing-based races, level hierarchies all clicked with the right instinct.
- Asks clarifying questions instead of guessing blind, and draws on real personal-project experience (Redis/TTL) when it's relevant.

**Areas to review before the real interview:**
- Distinguish "reduces how often a bug happens" (debouncing) from "actually closes the race" (an ignore-flag or `AbortController` inside a `useEffect` cleanup function) — these are not the same fix.
- The real trade-off of TTL-based caching is a staleness window, not extra CPU cost from checking timestamps.
- Local-vs-production reproducibility reasoning for timing-dependent bugs runs backwards right now: production has *more* timing variance than a fast, consistent dev machine, not less — that's usually why a race only shows up there.
- Get familiar with Spring Boot Actuator (`/actuator/loggers`, health, metrics) as the default toolbox for inspecting/managing a running app without a redeploy.
