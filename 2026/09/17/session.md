# Mock Interview — Junior Backend Developer
**Date:** 2026-09-17

**Categories covered (3-question session, date-based seed since no timestamp was available: seed = 17×137 + 9×31 = 2608, start=9, step=1; sequence: System Design & Architecture Basics → Tooling: Git, Build & Environment → Core Java & CS Fundamentals; opening category (9) differs from 09-13's opener (1) per the no-repeat-opener rule; all three sub-topics were checked against every prior session's knowledge summary to avoid reusing a scenario/mechanism already asked — System Design's formula pick (URL shortener / DRY vs. premature abstraction) had already been covered twice (CRUD schema checklist, bottleneck reasoning), so it shifted to the untested angle in that category, vertical vs. horizontal scaling and what a load balancer actually does; Tooling's formula pick (commit hygiene) had already been covered in full, so it shifted to the untested angle, what a reviewer looks for from the reviewer's seat rather than the author's; Core Java's formula pick (exceptions) had already been covered twice, most recently 09-13, so it shifted to the untested angle, ArrayList vs. LinkedList / HashMap vs. TreeMap):** System Design & Architecture Basics, Tooling: Git, Build & Environment, Core Java & CS Fundamentals

---

## Q1 — System Design & Architecture Basics

**Question:** Your team runs one instance of an image-processing service — it accepts an upload, resizes it, and returns the result synchronously. Under load, p95 response time is climbing, and CPU on that box is pegged. Your tech lead sketches two options on the whiteboard: bump the instance from 2 vCPU/4GB up to 8 vCPU/16GB, or run three copies of the current instance behind a load balancer.

Walk me through both options — what's actually different about them — and then tell me: once there are three instances behind a load balancer, how does it decide which instance handles a given incoming request, and what's one thing that has to be true about how your app is written for that setup to work correctly?

**Candidate's answer:**
> the p95 response time is climbing so that the response time is higher as time so in both options
> the first one is vertical scaling we use the higher CPU for this the other is horizontal scaling . It have many instance of the server
>
> i think it will make a request to instance that is it healthy or not if not it will route the request to another instance
>
> i think my app need to expose the port for this

**Score: 5/10**

**Strong:**
- Vertical vs. horizontal correctly distinguished: bumping CPU/RAM on one box vs. running multiple copies of the server.
- Correctly reached for health checks as part of what a load balancer does.

**Missing:**
- Health checks explain what happens when an instance is *down* — they don't explain how the load balancer picks *which* instance handles a normal request when all three are healthy. That's a separate mechanism: round-robin (cycle through instances in order) or least-connections (send to whichever instance has the fewest in-flight requests).
- "Expose the port" isn't the requirement horizontal scaling imposes — every instance already listens on a port regardless of instance count. The actual requirement is **statelessness**: the load balancer can route any request to any instance, so nothing request-specific can live only in one instance's memory (a session in a local `HashMap`, an in-memory cache). State needs to live somewhere all instances share (Redis, the database).

**Tip:** When a design question mentions "multiple instances," ask "what would break if request #2 from the same user landed on a *different* instance than request #1?" — that's usually where the real requirement is hiding.

---

## Q2 — Tooling: Git, Build & Environment

**Question:** You're reviewing a teammate's PR. It's a 40-line diff, CI is green, and at a glance the code looks clean and does what the ticket asked. What do you actually check before approving it — beyond "does it look fine and does CI pass"?

**Candidate's answer:** (after one clarifying push-back and one hint)
> it still vague does it code change relate to one thing or not
>
> *(hint given: CI only tells you the tests that already exist all pass — think about what that does NOT tell you)*
>
> oh i will check if another code of logic have but don't ahev any test case to test about this

**Score: 5/10**

**Strong:**
- Named PR cohesion (does this diff relate to one thing) unprompted.
- Once hinted, correctly landed on the core insight: CI only proves the tests that exist pass — it says nothing about logic that has no test covering it at all.

**Missing:**
- A hint was needed to reach "untested logic." A few more angles CI structurally can't check: does the code actually implement what the ticket asked (clean, tested code can still solve the wrong problem); security gaps (a new endpoint missing an auth check, a hardcoded secret, unvalidated input — nothing fails a test that was never written); architectural fit (business logic in a controller instead of a service, or duplicating an existing utility) — both compile and pass tests fine.
- One-line version to remember: CI answers "does this satisfy the tests we already wrote" — reviewing is everything CI structurally *can't* check.

**Tip:** Before approving any green PR, ask "if this were subtly wrong, would CI have caught it?" — if no, that's where review attention should go.

---

## Q3 — Core Java & CS Fundamentals

**Question:** You're building a service that keeps a running list of the last 500,000 order IDs processed today, and for every new incoming order you need to check "have we already seen this ID?" before processing it. A teammate's draft uses a `LinkedList<Long>` for this list, and calls `.contains(id)` on it for every check.

What's wrong with that choice, and what would you use instead?

**Candidate's answer:** (after two Socratic nudges, no hint text given outright)
> i think it will loop through the LinkedList to check it right ?
>
> i think i will use the Set because it can make the check is more faster than the LinkedList

**Score: 5/10**

**Strong:**
- Correctly reasoned that `LinkedList.contains()` traverses the list element by element.
- Landed on the right fix — a `Set` — rather than staying stuck on the wrong structure.

**Missing:**
- Took two nudges to assemble the chain: what `.contains()` does internally → why that's a problem at 500,000 entries → what to use instead. Worth walking that chain unprompted.
- "Set" is directionally right but imprecise — specifically a **`HashSet`**. A `TreeSet` is also a `Set` but still O(log n), not O(1); the speed-up comes from `HashSet`'s hashing/bucket mechanism (same as `HashMap`), not from being a `Set` per se.
- Precise cost: `LinkedList.contains()` is O(n) — up to 500,000 comparisons for a single duplicate check late in the day. `HashSet.contains()` is O(1) average — one hash computation, one bucket lookup, regardless of how many IDs are already in the set.

**Tip:** Whenever `.contains()` shows up in a hot path, ask "what's this backed by — a list (linear scan) or a hash-based structure (near-constant time)?" before anything else.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | System Design & Architecture Basics | 5/10 |
| 2 | Tooling: Git, Build & Environment | 5/10 |
| 3 | Core Java & CS Fundamentals | 5/10 |

**Overall: 5.0/10**

**Strengths:**
- Reliably lands on the *right* fundamental concept first (vertical vs. horizontal scaling, PR cohesion, `.contains()` traversal, `Set` as the fix) — the instincts are sound.
- Willing to reason out loud and follow a nudge to the correct conclusion rather than freezing or guessing randomly.

**Areas to review before the real interview:**
- Load balancer mechanics: name the actual routing decision (round-robin, least-connections) separately from health-check failover, and know that horizontal scaling requires the app to be stateless.
- A fuller PR review checklist beyond "does it pass tests": requirement match, security gaps, architectural fit.
- Data structure precision: `HashSet`/`HashMap` are fast because of hashing into buckets (O(1) average) — a generic "Set" or "Map" isn't automatically fast (`TreeSet`/`TreeMap` are O(log n)); name the specific structure and mechanism, not just the category.
