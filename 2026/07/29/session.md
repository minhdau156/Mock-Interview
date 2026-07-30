# Mock Interview — Junior Backend Developer
**Date:** 2026-07-29

**Categories covered (seed=4190, start=1, step=9):** Core Java & CS Fundamentals → Tooling: Git, Build & Environment → System Design & Architecture Basics → Web Security & Frontend Basics → Testing & Debugging

*Note: this session deliberately targeted sub-topics NOT already documented in KNOWLEDGE_SUMMARY.md's "Knowledge Base by Category" section, per candidate request, rather than the exact per-question angle the seed formula would otherwise select.*

---

## Q1 — Core Java & CS Fundamentals

**Question:** You've built a `Set<Customer>` to dedupe customer records coming in from two data feeds before writing them to your DB. You override `equals()` to compare by `customerId`, but you never touch `hashCode()` — IDE just left the default `Object.hashCode()` in place. In testing, the same customer (same `customerId`, loaded twice from the two feeds) shows up **twice** in the `Set` instead of being deduped once. What's going wrong, and why does forgetting `hashCode()` break this even though `equals()` is correct?

**Answer (after one hint about how `HashSet.add()` finds duplicates):**
> HashSet.add() uses hashCode to get a number and place it in the Set; without overriding hashCode, it gets added twice

**Score: 5/10**

**Strong:**
- Correctly landed on the core idea: `hashCode()` is what determines where an object gets placed, and leaving it un-overridden is what causes the duplicate.

**Missing:**
- The full mechanism: `HashSet` doesn't scan every existing element with `.equals()`. It first calls `hashCode()` on the incoming object to compute a **bucket index** (`hash % numBuckets`), then only compares `.equals()` against objects already in that same bucket. Different buckets → `.equals()` never even runs.
- Why it happens here specifically: default `Object.hashCode()` is identity-based (roughly memory address), not based on `customerId`. Two separate `Customer` instances with the same `customerId` are different objects → different default hash codes → almost certainly different buckets.
- The name for this rule: the **`hashCode`/`equals` contract** — "if two objects are equal via `.equals()`, they must return the same `hashCode()`." Overriding one without the other silently violates it.

**Tip:** Whenever you override `equals()`, generate `hashCode()` in the same action (most IDEs do both together) — never one alone.

---

## Q2 — Tooling: Git, Build & Environment

**Question:** You're mid-feature on a branch that's a few days old. You pull the latest `main` to merge it in, and Git stops with a conflict in `OrderService.java` — a teammate refactored the same method you've been modifying. What do you actually do, step by step, to resolve it correctly — specifically, how do you make sure you don't accidentally throw away either your teammate's refactor or your own changes?

**Answer:**
> check who pushed the file more recently, then discuss with the teammate to choose between the two versions

**Score: 4/10**

**Strong:**
- Good instinct to loop in the teammate to understand the intent behind their change.

**Missing:**
- Skipped the actual mechanics: open the file, read the `<<<<<<< HEAD` / `=======` / `>>>>>>>` markers, manually edit them into one correct version, delete the markers.
- Key correction: this is almost never "choose between 2 options." That framing is the exact mistake the question was probing for — picking one side wholesale is how you silently discard the other person's real work. Usually both changes need to coexist (teammate's restructure + your addition), not an either/or pick.
- After merging the logic: test/run it, `git add <file>`, then `git commit` (merge) or `git rebase --continue` (rebase).
- IDE 3-way merge view (yours/theirs/result) is safer than eyeballing raw markers.

**Tip:** Run `git log -p <file>` or `git blame` on the conflicting lines before messaging the teammate — often makes "combine both" obvious without a conversation.

---

## Q3 — System Design & Architecture Basics

**Question:** A "place order" endpoint needs to (1) charge the customer's card and (2) send a confirmation email. A teammate suggests calling the email service the same way as the payment service — a direct synchronous call. Would you call the email service the same way, or handle it differently? Walk me through your reasoning.

**Answer:**
> disagreed with synchronous email calls (latency/server load under high traffic); proposed using async — HttpClient async in Java or async/await in JS

**Score: 5/10**

**Strong:**
- Correct conclusion: don't make the order endpoint synchronously wait on the email service.
- Correctly distinguished payment and email shouldn't be treated the same way.

**Missing:**
- `async/await`/async `HttpClient` is still a **direct call** to the email service — just non-blocking on the calling thread. The order endpoint still needs to deal with the email service's failure/timeout in that request's code path. That's not the same as a message queue.
- The real distinction: direct call (sync or async style — caller and callee coupled) vs. **message queue** (publish an "OrderPlaced" event, return immediately; a separate consumer sends the email whenever it can, tolerating the email service being down entirely).
- Decision rule: "does the caller need the result before responding to the user?" Payment: yes (must confirm charge). Email: no.
- Missed: queues also give free retries for transient failures, instead of a fire-and-forget async call silently swallowing them.

**Tip:** Ask "async from the caller's *thread*, or decoupled from the caller's *request lifecycle* entirely?" — that's the difference between `async/await` and a real message queue.

---

## Q4 — Web Security & Frontend Basics

**Question:**
```java
String query = "SELECT * FROM products WHERE name = '" + request.getParameter("name") + "'";
Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(query);
```
It works fine in manual testing. What's actually wrong with this code, what could a malicious user do with it, and how would you fix it?

**Answer (after one hint about a `'` breaking out of the string literal):**
> SQL injection; attacker could inject a JOIN to pull data from the orders/users tables; fix with SQL parameters plus request validation

**Score: 6/10**

**Strong:**
- Correctly named SQL injection via string concatenation.
- Gave a concrete, non-trivial attack: injecting a `JOIN` to exfiltrate unrelated data (orders/users tables) — shows understanding of impact, not just the label.
- Named the correct fix direction: parameterized queries.

**Missing:**
- The actual Java mechanism: `PreparedStatement` with `?` placeholders and `.setString(...)`, not just "SQL parameter" as a vague term.
- **Why** it fixes it: the SQL structure is compiled separately from the bound value — the database never re-parses user input as SQL syntax, no matter what characters it contains.
- Correction: request validation is fine as defense-in-depth but is **not** a substitute for parameterization — blacklist-style validation is routinely bypassed. Parameterized queries are what actually close the hole.

**Tip:** Flag `"... WHERE x = '" + var + "'"` string concatenation in review immediately, regardless of how safe manual test input looks.

---

## Q5 — Testing & Debugging

**Question:**
```java
@Test
void appliesDiscount() {
    ProductRepository mockRepo = mock(ProductRepository.class);
    PricingEngine mockPricing = mock(PricingEngine.class);
    DiscountCalculator mockCalculator = mock(DiscountCalculator.class);
    when(mockRepo.findById(1L)).thenReturn(product);
    when(mockPricing.getBasePrice(product)).thenReturn(100.0);
    when(mockCalculator.calculate(100.0, 0.1)).thenReturn(90.0);
    DiscountService service = new DiscountService(mockRepo, mockPricing, mockCalculator);
    double result = service.applyDiscount(1L, 0.1);
    assertEquals(90.0, result);
}
```
Something about this test bugs you even though it passes. Is it verifying anything useful about `DiscountService`, or something else?

**Answer (after a hint and a full concept explanation):**
> concluded the test isn't really testing anything meaningful, but couldn't independently articulate the mechanism or a fix

**Score: 3/10**

**Strong:**
- Reached the right conclusion in the end, restated in own words after support.

**Missing:**
- Needed both a hint and a full walkthrough before restating the mechanism — the deliverable is being able to say it unprompted: every mock is stubbed to return exactly the value the assertion checks, so the test verifies Mockito echoed its own stub, not that `DiscountService` computed anything.
- No proposed fix given: either `verify()` the orchestration (correct arguments passed to each collaborator), or stop mocking every dependency and test the real discount math directly.
- Vocabulary to have ready: this is the **over-mocking** / "testing the mock, not the code" anti-pattern.

**Tip:** If every dependency is mocked and the stub's return value matches the assertion, ask "what real behavior is left to fail if I break something?" — if none, the test isn't testing that class.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Core Java & CS Fundamentals — `HashSet` dedup miss from unoverridden `hashCode()` | 5/10 |
| 2 | Tooling: Git, Build & Environment — resolving a merge conflict without losing work | 4/10 |
| 3 | System Design & Architecture Basics — sync call vs message queue for email | 5/10 |
| 4 | Web Security & Frontend Basics — SQL injection via string concatenation | 6/10 |
| 5 | Testing & Debugging — over-mocking / testing the mock instead of the code | 3/10 |

**Overall: 4.6/10**

**Strengths:**
- Gave a genuinely concrete, non-trivial exploitation scenario for SQL injection (JOIN-based data exfiltration across tables) rather than a generic "it's insecure."
- Correct end conclusions on both the `HashSet` and sync/async questions, even where the underlying mechanism needed a hint to fully articulate.
- Willing to ask for help rather than guess wildly or freeze — useful self-awareness, though this session needed more hand-holding than average (3 of 5 questions used a hint or full explanation).

**Areas to review before the real interview:**
- Merge conflict resolution — the core misconception this session: resolving a conflict means combining both changes into one correct version, not picking one side by discussion. Review the actual mechanics: reading conflict markers, merging both sides, testing, `git add`, then `git commit`/`rebase --continue`.
- Sync vs. async architecture vocabulary — `async/await` or a non-blocking `HttpClient` call is still a direct call to the downstream service (coupled to its availability); a message queue is a different, stronger form of decoupling. Don't conflate the two.
- Mocking anti-patterns — the weakest question this session, needing the most support. Review Mockito fundamentals: when a mock is testing real orchestration (via `verify()`) vs. when it's just echoing back its own stub.
- `hashCode`/`equals` contract — solidify the exact mechanism (bucket index from `hashCode()`, `equals()` only checked within that bucket) so it can be explained without a hint next time.
- SQL injection fix mechanics — know why `PreparedStatement` works (query structure and data sent to the DB separately) and that input validation is not a substitute for parameterization.
