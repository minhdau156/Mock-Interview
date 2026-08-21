# Mock Interview — Junior Backend Developer
**Date:** 2026-08-21

**Categories covered (3-question session, date-based seed (no timestamp available): seed = 21×137 + 8×31 = 3125, start=6, step=1; sequence from start: Concurrency & Locking → Testing & Debugging → Web Security & Frontend Basics; opening category (6) differs from 08/20's opener (9) per the no-repeat-opener rule; Q1 and Q2 substitute a fresh sub-topic within their category because the formula's raw pick landed on an already-used angle (in-memory counter race, and arrange/act/assert flaky test), same substitution approach as 08/20's notes; Q3's formula pick landed cleanly on an unused angle already; all three scenarios were checked against every prior session back to 07/04 to avoid reusing a scenario/code snippet already asked):** Concurrency & Locking, Testing & Debugging, Web Security & Frontend Basics

---

## Q1 — Concurrency & Locking

**Question:** Your team's order service used to run on a single instance. It just got scaled to 3 instances behind a load balancer for redundancy. There's a `@Scheduled` job that runs every night at 2 AM and emails customers who abandoned their cart. Ever since the scale-out, customers report getting the same reminder email three times each morning.

A teammate's fix: `synchronized` on the scheduled method.

Walk me through: would that actually stop the triple-send? And separately — what's a real way to make sure only one of the three instances runs this job each night, and what should happen if that one instance crashes halfway through the job?

**Candidate's answer (after a clarifying question on whether `@Scheduled` runs on a thread):**
> "i think the synchronized can not stop this because it just work in multithreading in this case the @Scheduled is setup to send the email at 2 AM so it can stop the duplicate email / and each thread belong to each instance of server so it can not handle the multiple email that are duplicate to user / so i think i need to use a distributed db like redis when send it i will create a key value and at to redis so each instance need to check before send"

**Feedback given:**
- Strong: correctly diagnosed why `synchronized` fails — it only coordinates threads inside one JVM; each of the 3 instances has its own JVM/thread pool/lock, so all three still fire at 2 AM.
- Strong: landed on the right category of fix unprompted — an external coordination point (Redis), not something inside one process.
- Missing: the actual atomic mechanism — Redis `SET lock_key <value> NX PX <ttl>`, a single atomic command that only succeeds if the key doesn't already exist, avoiding a check-then-set race.
- Missing: did not address the crash scenario at all — without a TTL, a crashed lock-holder holds the lock forever and the job never runs again; `PX <ttl>` gives auto-expiry so the next scheduled run can acquire it fresh.
- Missing: the term "distributed locking" as the named pattern.

**Score: 6/10**

---

## Q2 — Testing & Debugging

**Question:** You're debugging a checkout flow that chains five internal calls in sequence: `validateCart → applyPricing → reserveInventory → chargePayment → sendConfirmation`. Customers occasionally get a generic "Checkout failed, please try again" error, with no indication of which step it came from. A teammate's plan: "I'll just sprinkle `System.out.println` after every single line across all five methods and re-deploy to see what prints."

Is that a reasonable way to track this down? Walk me through how you'd actually narrow down which step is failing, and what tools or techniques you'd reach for instead of scattering prints everywhere.

**Candidate's answer (after two clarifying questions confirming a specific exception is likely being swallowed into the generic message, and one explicit hint):**
> "i think the first thing is i need to find some exception that swallow so that handle it in the GlobalHandleException so it can log the specific message in it / the second question that i have ever use any tool like this in my project so but i just use the loki grafana and tempo"

**Feedback given:**
- Strong: good instinct on the swallowed exception — fixing the `GlobalExceptionHandler`/`@ControllerAdvice` to log the specific exception means future occurrences are visible without redeploying.
- Strong: named real observability tooling — Loki, Tempo, Grafana — a genuinely production-grade answer for isolating a failing step in a live service without redeploying.
- Missing: stayed at the tool-name level — didn't explain that a trace gives a span per step, and you'd look at where the trace stops/errors to know exactly which step failed.
- Missing: didn't address the debugger/breakpoint angle from the hint at all.
- Missing: the general principle of binary-search debugging — narrowing the failure by checking a midpoint rather than instrumenting everything at once.
- Missing: needed two clarifying rounds plus an explicit hint to reach this answer.

**Score: 5/10**

---

## Q3 — Web Security & Frontend Basics

**Question:** Your team's React admin panel has an "Apply Employee Discount" button:

```jsx
{user.role === 'admin' && (
  <button onClick={applyEmployeeDiscount}>Apply Employee Discount</button>
)}
```

A teammate reviewing the PR says: "Nice, that's secure — a regular employee won't even see the button, so they can't use it."

Do you agree that this is secure? What could a non-admin user still do here, and what actually needs to happen for this to be safe?

**Candidate's answer:** (asked whether a non-admin could tamper with `user.role` client-side — confirmed yes, and nudged toward thinking about what `onClick` actually does under the hood; asked for a hint, received one pointing at calling the backend endpoint directly with curl/Postman; then said "i don;t know can you give me the answer" without attempting a guess)

**Feedback given:**
- Strong: correct partial instinct that `user.role` is client-side state and could in principle be tampered with.
- Missing: the core answer — hiding the button is a UX decision, not a security boundary. Any user can call the backend endpoint (e.g. `POST /api/discounts/employee`) directly via curl/Postman/devtools, completely bypassing the button and the React role check.
- Missing: the real fix — the backend endpoint itself must independently verify the caller's role (e.g. Spring Security `@PreAuthorize("hasRole('ADMIN')")` against a signature-verified JWT claim/session), since the client cannot forge that.
- Missing: the general rule — never trust client input, including anything controlling appearance only (hidden buttons, disabled inputs, grayed-out menus); the frontend check is fine to keep for UX but must never be the only check.
- Missing: went straight from "I don't know" to "give me the answer" after one hint, without venturing a guess.

**Score: 3/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Concurrency & Locking — distributed lock (Redis `SET NX PX`) for a `@Scheduled` job across instances | 6/10 |
| 2 | Testing & Debugging — narrowing down a failing step in a 5-call chain (tracing/logging vs. scattering prints) | 5/10 |
| 3 | Web Security & Frontend Basics — hiding a button isn't security; server-side authorization | 3/10 |

**Overall: 4.7/10**

**Strengths:**
- Consistently asks sharp clarifying questions and shows correct partial instincts early (synchronized is JVM-local, client role state is tamperable, exceptions might be swallowed somewhere).
- When landing on a mechanism, it's often genuinely correct and even production-grade (Redis distributed lock, real observability stack with Loki/Tempo/Grafana) — the instincts are there.

**Areas to review before the real interview:**
- Commit to a guess before asking for the answer. All three questions needed heavy hinting, and Q3 ended with "just give me the answer" after a single hint. Interviewers score reasoning attempts far higher than silence — "I think X because Y, but I'm not fully sure" is a strong answer even when wrong.
- Client-side checks are never a security boundary — a hidden button, a disabled field, or a JS role check only controls what's shown; the server must independently re-check authorization for every sensitive action.
- Distributed locks need a TTL/expiry (Redis `SET NX PX`) — without one, a crashed lock-holder blocks the job forever. Practice stating the full atomic mechanism, not just "use Redis."
- Narrowing down a multi-step failure: practice articulating the why behind a tool (a trace shows a span per step, so you can see exactly where it broke) rather than just naming the tool.
