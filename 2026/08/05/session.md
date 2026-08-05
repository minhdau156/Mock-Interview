# Mock Interview — Junior Backend Developer
**Date:** 2026-08-05

**Categories this session (3-question format):** Spring Boot & JPA/Hibernate, Testing & Debugging, Tooling: Git, Build & Environment
**Seed:** 933 (date-only) — start index 4, step 3

---

## Question 1 — Spring Boot & JPA/Hibernate

**Q:** You're reviewing a teammate's PR for `GET /orders/{id}`. Their unit test mocks the repository and passes fine. But when they hit the real endpoint manually (real DB, real HTTP call), it throws:

```
org.hibernate.LazyInitializationException: could not initialize proxy [Customer#42] - no Session
```

...right at the line in the controller where the response is being built, doing `order.getCustomer().getName()`.

Your teammate is confused: "The service method returned fine — the exception happens *after* that, when it's just building the JSON response. How does that even fail?"

What's actually going on here, and how would you fix it? Give at least two different valid fixes.

**Candidate's process:**
- Asked for an explanation of what `LazyInitializationException` means (hint 1 given: generic mechanism — accessing a not-yet-loaded association after the Session that would fetch it is closed).
- Asked for a hint on when the Session closes (hint 2 given: pointed at which annotation manages the transaction boundary and which layer it's usually on).
- Initially answered "the repository" (incorrect — corrected to "usually the service layer").
- Final answer: correctly traced `@Transactional` on the service → session closes when service method returns → controller's later lazy access fails. Proposed two fixes: (1) fetch the customer separately via id/another service call, (2) use `JOIN FETCH` in the query to eagerly load the customer and avoid the lazy call.

**Score: 6/10**

**Strong:**
- After the hints, correctly traced the full mechanism: `@Transactional` on the service opens/manages the Hibernate session, and it closes the instant the service method returns — so when the controller later touches `order.getCustomer().getName()`, there's no open session left to lazily fetch `Customer` through, hence the exception.
- Fix #2 (`JOIN FETCH`) is exactly the standard, most common fix — eagerly load the association *inside* the still-open transaction so no lazy call happens afterward.
- Fix #1 (fetch by id via a separate service call) is a workable alternative direction.

**Missing:**
- Needed two hints to reach the mechanism — worth stating unprompted next time: "the entity/proxy becomes detached the moment the transaction commits and the session closes; any association not yet initialized *inside* that transaction can never be lazily initialized afterward."
- Fix #1 as phrased sounds like it assumes touching `order.getCustomer()` at all is dangerous — but in JPA, `order.getCustomer().getId()` (the FK scalar) is actually safe even on a lazy, closed-session proxy; only accessing a *non-id* field like `getName()` triggers the failing fetch. Worth naming that distinction explicitly.
- The most common production-safe alternative wasn't mentioned: map the entity to a **DTO inside the service method** (while the transaction is still open) and return the DTO — then the controller only ever serializes plain fields, never a lazy proxy, regardless of which associations get touched later.
- Related anti-pattern worth knowing by name: **"Open Session in View"** — extending the Hibernate session across the whole HTTP request "fixes" this exception by masking it, but hides N+1 queries at render time; Spring recommends disabling it (`spring.jpa.open-in-view=false`) and fixing the real cause instead.

**Tip:**
- Reflex: the moment an entity leaves its `@Transactional` method (returned to a controller, cached, handed to another thread), treat every non-eagerly-loaded association as off-limits — fetch it explicitly inside the transaction, or convert to a DTO before returning.

---

## Question 2 — Testing & Debugging

**Q:** A teammate opens a PR adding a `PriceCalculator.calculate()` method. Their test for it is annotated `@SpringBootTest` — it boots the entire Spring context, wires in the real `PriceRepository` hitting a real test database, and asserts the final price. It passes, but it takes about 8 seconds to run by itself. Your tech lead notices CI now takes 15 minutes for ~200 tests like this one and asks why.

What's actually being tested here versus what should be tested at this level, and how would you restructure it? Also give the general rule you'd use to decide "does this deserve a unit test or an integration test."

**Candidate's answer:** Identified that `@SpringBootTest` + real DB wastes time when the goal is testing business logic. Proposed restructuring to `@ExtendWith(MockitoExtension.class)` with a mocked `PriceRepository`, verifying the method's logic, its calls to the repository, and its exception paths without booting the app or hitting a real DB. Initial rule given: "if the method has a lot of business logic, use unit test." Refined unprompted afterward: "if there's a DB call or API call, I'll use integration test."

**Score: 7/10**

**Strong:**
- Correctly diagnosed the core problem: `@SpringBootTest` boots the full context and hits a real DB — unnecessary overhead for testing pure calculation logic, and exactly why CI got slow.
- Correct restructuring: swap to `@ExtendWith(MockitoExtension.class)`, mock `PriceRepository` (and any other collaborator), test `calculate()` in isolation — that's the standard fix.
- Good instinct on what a unit test should actually check: internal logic/branching, whether it calls the repository correctly (`verify()`), and exception paths — none of which need a real DB or app context.
- Self-corrected the general rule unprompted, refining from "amount of logic" to "does it involve a DB/API call" — closer to the right lens.

**Missing:**
- Missing the shape/vocabulary of the **testing pyramid** itself: many fast, isolated unit tests at the base, fewer integration tests validating real boundaries, very few end-to-end tests at the top. The fix isn't "stop using `@SpringBootTest`" — it's "use it sparingly for the few things that genuinely need a real DB/context, and keep pure-logic classes like `PriceCalculator` on fast unit tests."
- Didn't mention that the repository query itself still deserves its own (much smaller number of) integration test somewhere — mocking it here doesn't mean nobody verifies the real query works, it just means that verification belongs in a different, narrower test.

**Tip:**
- Reflex for any test: ask "which collaborators am I trusting vs. which one am I actually verifying?" Everything trusted gets mocked; only the thing under test runs for real. If the honest answer is "all of them, including the DB," that's an integration test — fine, just don't write 200 of them.

---

## Question 3 — Tooling: Git, Build & Environment

**Q:** You've been working on a feature branch for four days. Five other commits landed on `main` from teammates in that time. Before opening your PR, you want your branch to include those changes.

Would you `git merge main` into your feature branch, or `git rebase main`? What does each actually do to your commit history, and which would you pick here? Is there a situation where you'd deliberately avoid rebase even though it'd otherwise be your preference?

**Candidate's process:**
- Asked for the mechanics of merge vs. rebase from scratch (hint 1: explained merge creates a two-parent merge commit, non-linear history; rebase replays commits on top of `main` with new hashes, linear history).
- Picked rebase, but reasoned it was because rebase "keeps history of who did the change" — a misconception (both merge and rebase preserve authorship). Corrected.
- Pointed toward the force-push/shared-branch danger (hint 2: asked what happens if a teammate already pulled and built on the original commits before a rebase + force-push).
- Final answer: "it will conflict" — in the right direction but not precise about the actual mechanism.

**Score: 4/10**

**Strong:**
- Correctly picked rebase as the practical choice for updating a private feature branch before a PR.
- After a nudge, landed in the right neighborhood ("conflict") for the danger scenario — on the right track, just not precise.

**Missing:**
- Needed a full from-scratch explanation of merge vs. rebase mechanics — this is daily-use tooling and should be recallable without a hint.
- The stated reason for preferring rebase ("so I know who did it") was a genuine misconception — both merge and rebase preserve commit authorship (`git blame` works fine either way). The real reasons to prefer rebase here are a cleaner, linear history and an easier-to-read PR diff, not authorship tracking.
- "It will conflict" undersells the actual danger. It's not a routine merge conflict — it's this: rebase gives every one of your commits a **new hash**, even though the content is identical. If a teammate already pulled your branch and built their own commits on top of your *original* hashes, force-pushing your rebased branch pulls the rug out from under them — their branch now points at commits that no longer exist upstream, and reconciling it risks duplicated or lost work. Rule to memorize: **never rebase (and force-push) a branch other people have already pulled and built on** — rebase freely on a private branch only you use; once it's shared, prefer merge (or coordinate the rebase explicitly with everyone on it).

**Tip:**
- Reflex before any rebase: "has anyone else pulled this branch?" If yes, don't rebase it — or if you must, make sure everyone re-syncs and knows it's coming.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Spring Boot & JPA/Hibernate — `LazyInitializationException` from a detached entity | 6/10 |
| 2 | Testing & Debugging — testing pyramid, unit vs. integration | 7/10 |
| 3 | Tooling: Git, Build & Environment — merge vs. rebase | 4/10 |

**Overall: 5.7/10**

**Strengths:**
- Good diagnostic instinct on the testing-pyramid question — correctly identified why `@SpringBootTest` was the wrong tool here and proposed the standard `MockitoExtension` + mocked-repository fix, then unprompted refined the rule to "DB/API call in the test → integration test."
- Once given the mechanism, correctly reasoned through the full `LazyInitializationException` chain (transaction closes → lazy proxy access fails) and produced two workable directions to fix it.
- Willing to reason out loud, ask targeted clarifying questions, and take hints rather than freeze or guess wildly.

**Areas to review before the real interview:**
- Merge vs. rebase mechanics and the "never rebase a shared/already-pulled branch" rule — needed a full explanation from scratch; this is core daily tooling and the weakest showing of the session.
- `LazyInitializationException` / entity detachment — needed two hints to connect `@Transactional`'s boundary to why the session is closed by the time the controller runs; review "an entity detaches the instant its owning transaction commits."
- DTO projection and "Open Session in View" as the standard production-grade answers for lazy-loading-across-layers problems — neither came up.
- Testing pyramid vocabulary — frame unit vs. integration around *what boundary is being verified*, not "how much logic," and know the pyramid shape (many fast unit tests, fewer integration tests, very few end-to-end).
