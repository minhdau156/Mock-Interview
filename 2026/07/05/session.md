# Mock Interview — Junior Backend Developer
**Date:** 2026-07-05

**Categories this session:** Databases (Indexing) → Tooling (Environments/Secrets) → Testing & Debugging → Spring Boot & JPA/Hibernate → Core Java (Exceptions)
**Seed:** 902 (start=3, step=7)

---

## Question 1 — Databases: SQL, Indexing & Data Modeling

**Q:** You're reviewing a teammate's PR. They added this query to a `users` table with 2 million rows:
```sql
SELECT * FROM users WHERE email = 'someone@example.com';
```
There's already an index on the `email` column, but they mention it's still taking ~800ms and ask you why an index isn't helping. You look closer and see the column was actually created as `email VARCHAR(255)` with the index on `email`, but the application code that populates it sometimes stores emails in mixed case, and the query above uses whatever case the user typed at login.

What's going on here, and what would you tell them to check or change?

**Candidate's answer (after 2 hints):** "i think it sort the email column so it will work good it will sort base on alphabetically so i think it will work correctly"

**Score: 3/10**

**Strong:**
- Correctly connects an index to sorting/ordering the column — the right starting intuition (B-tree keeping values in order for fast lookups).

**Missing:**
- The real bug: to handle inconsistent casing, a common (wrong) fix is wrapping the column in a function, e.g. `WHERE UPPER(email) = UPPER('...')`. Applying a function to an indexed column defeats the index — the database indexed raw values, not their transformed form — forcing a full table scan.
- Correct fix: normalize casing at write time (store lowercase before insert/update) so the query can compare directly and use the index as-is. (Alternative: case-insensitive collation/type, or a functional index on `UPPER(email)`.)
- Should run `EXPLAIN` to confirm a sequential/full scan vs an index lookup rather than guessing.

**Tip:** Any `WHERE SOME_FUNCTION(column) = ...` in a slow query is a red flag for index defeat. (Two hints needed — revisit how indexes match values, not just that they sort them.)

---

## Question 2 — Tooling: Git, Build & Environment

**Q:** A teammate is onboarding onto your team's Spring Boot project. To get it running quickly, they commit a `.env` file into the repo containing the real staging database password and a third-party API key — with a comment saying `// works on my machine now, just pull this`. Everyone who clones the repo can now run the app immediately without asking anyone for credentials.

You're reviewing this in a PR. What's wrong with this approach, and what would you tell them to do instead?

**Candidate's answer:** "i think it can have modify something that existing in the project and can cause som bug problem, they can you the API key in the third party to do more things. it very dangerous, i think i will change the API key and don't add the .env file to git"

**Score: 6/10**

**Strong:**
- Correctly identified the core danger: a real API key/password in the repo can be misused by anyone with access to that history.
- Fix direction is right: rotate/revoke the exposed key, stop committing `.env`.

**Missing:**
- Why rotation is critical: once committed, a secret stays in git history forever even after later deletion — so rotation is mandatory, not optional.
- Concrete mechanism: add `.env` to `.gitignore` so it can't be accidentally re-committed.
- Onboarding-friendly fix: commit a `.env.example` with placeholder values.
- Where real secrets should live: injected as env vars at runtime via CI/CD secret store or secret manager, never hardcoded in the repo.

**Tip:** Add `.env` to `.gitignore` on day one; consider secret-scanning tools (GitHub secret scanning, gitleaks) in CI.

---

## Question 3 — Testing & Debugging

**Q:** You're investigating a bug ticket that says: "Sometimes when a user submits the checkout form twice quickly, they get charged twice." You can't reproduce it on your first few tries clicking the button normally, and the logs from the report are a huge wall of text with no obvious error.

Walk me through how you'd actually go about tracking this down — what's your first move, and how do you narrow it down from "sometimes" to something you can reliably trigger and observe?

**Candidate's answer (after 1 hint):** "i think i will use the id of the request to trace the error and see the log"

**Score: 4/10**

**Strong:**
- Filtering logs by a request/correlation ID to isolate relevant lines out of a huge log file is a genuinely correct and useful technique.

**Missing:**
- Doesn't address the harder half: reliably triggering an intermittent, timing-dependent bug. Manual double-clicking rarely hits the exact race window — should force the race (script/Postman firing two near-simultaneous requests, or a debugger breakpoint pausing the first request mid-flight).
- No mention of log levels — the needed detail may only appear at DEBUG/TRACE, not the default INFO/WARN.
- No mention of bisecting the flow (binary-search debugging) to narrow where the duplicate charge is introduced.

**Tip:** For any "sometimes" bug, the first goal is turning it into an "always" bug you can trigger on demand.

---

## Question 4 — Spring Boot & JPA/Hibernate

**Q:** You're pairing with a junior teammate on a `PaymentService` class:
```java
public class PaymentService {
    private PaymentGateway gateway = new PaymentGateway();

    public void charge(Order order) {
        gateway.charge(order.getTotal());
    }
}
```
It compiles and works in their manual testing, but when you try to write a unit test for `PaymentService` that doesn't hit the real payment gateway, you get stuck — and separately, your tech lead comments "this won't work well as a Spring bean, please use constructor injection instead."

What's the problem with the class as written, and what would constructor injection actually change here?

**Candidate's answer:** "i think it create other PaymentGateway, in case this is the interface, it will be the other PaymentGateway, and you can not have the real gate way, when use the constructor injection it can change the paymentgate way easily and can not get stuck, and it do not depend to much to PaymentGateway class"

**Score: 6/10**

**Strong:**
- Correctly spotted that `new PaymentGateway()` hardcodes the dependency, blocking substitution of a mock for testing.
- Correctly understands constructor injection lets you supply a different `PaymentGateway` from outside.

**Missing:**
- Didn't connect it to "won't work well as a Spring bean" — Spring manages beans by providing their dependencies from the container; a self-constructed field is invisible to Spring, so it can't be swapped, overridden per profile, or lifecycle-managed.
- Constructor injection allows a `final` field — guaranteed non-null once constructed, and missing dependencies fail fast at startup instead of surfacing as a `NullPointerException` later.
- With constructor injection, testing is as simple as `new PaymentService(mockGateway)` — no Spring context or reflection needed.

**Tip:** Whenever a class does `new SomethingWithBehavior()` internally instead of receiving it, ask "how would I test this in isolation?"

---

## Question 5 — Core Java & CS Fundamentals

**Q:**
```java
public String readFirstLine(String path) {
    try {
        BufferedReader reader = new BufferedReader(new FileReader(path));
        return reader.readLine();
    } catch (IOException e) {
        return null;
    }
}
```
First, `IOException` is a checked exception, meaning the original method had to either catch it or declare `throws IOException` — explain why that's true for `IOException` but wouldn't be true if it threw a `NullPointerException` instead. Second, there's a bug related to resource handling — what is it, and how would you fix it?

**Candidate's answer (after 2 hints):** "i think it will be conflict when use NullPointerException"

**Score: 2/10**

**Strong:**
- Engaged with the question rather than disengaging, even without landing on the answer.

**Missing:**
- Checked vs unchecked is about class hierarchy: `IOException` extends `Exception` (not `RuntimeException`), so the compiler forces callers to catch or declare it — because checked exceptions typically represent recoverable, expected failures (missing file, network issue). `NullPointerException` extends `RuntimeException`; everything under `RuntimeException` is unchecked because these usually represent programming bugs that could occur anywhere, so mandating declaration everywhere would be unworkable.
- Didn't identify the actual bug: the `BufferedReader`/`FileReader` is never closed — a resource leak. Fix with try-with-resources:
```java
public String readFirstLine(String path) {
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        return reader.readLine();
    } catch (IOException e) {
        return null;
    }
}
```

**Tip:** Whenever a resource (file, stream, connection, reader) is opened with `new` inside a method, the reflex should be "does this need try-with-resources?" (Two hints needed — review exception hierarchy and try-with-resources before the real interview.)

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Databases: Indexing (index-defeating functions) | 3/10 |
| 2 | Tooling: Secrets & environment config | 6/10 |
| 3 | Testing & Debugging: reproducing intermittent bugs | 4/10 |
| 4 | Spring Boot: constructor injection / DI | 6/10 |
| 5 | Core Java: checked vs unchecked exceptions, try-with-resources | 2/10 |

**Overall: 4.2/10**

**Strengths:**
- Understands secrets shouldn't live in git and knows the core remediation (rotate + stop committing).
- Grasps the basic idea of dependency injection — hardcoding `new SomeDependency()` blocks substitution, constructor injection fixes that.
- Instinctively reaches for a request/correlation ID to isolate one request out of noisy logs.

**Areas to review before the real interview:**
- Indexing internals: why applying a function to an indexed column (e.g. `UPPER(email)`) prevents index use — and reading `EXPLAIN` output to confirm.
- Reproducing intermittent/race-condition bugs: forcing timing-dependent bugs to happen on demand instead of relying on manual repetition.
- Exception hierarchy: `Exception` vs `RuntimeException`, and why the compiler enforces checked exceptions but not unchecked ones.
- Resource management: try-with-resources as the default reflex whenever a `Reader`/`Stream`/`Connection` is opened.
- Confidence under uncertainty: several answers needed two hints before a partial answer emerged — practice stating a best guess with reasoning even when unsure, rather than waiting for more hints.
