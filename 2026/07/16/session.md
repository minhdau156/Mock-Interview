# Mock Interview — Junior Backend Developer
**Date:** 2026-07-16

**Categories covered (seed=2409, start=10, step=1):** Tooling: Git, Build & Environment → Core Java & CS Fundamentals → OOP, Design Patterns & LLD → Databases: SQL, Indexing & Data Modeling → Spring Boot & JPA/Hibernate

---

## Q1 — Tooling: Git, Build & Environment

**Question:** You're reviewing a teammate's pull request and see a single commit titled "fixes" that touches the auth middleware, renames three unrelated utility functions, and tweaks a CSS file. It all technically works. What do you say in the review, and why does it matter beyond just "style"?

**Answer:**
> proposed splitting into 3 commits by reason (fix: .../chore: rename .../chore: tweak CSS ...), reasoning that it allows reverting just the auth fix later without losing the CSS change

**Score: 7/10**

**Strong:**
- Correctly proposes splitting the commit by logical concern into `fix:`/`chore:`/`chore:`, which is exactly the right instinct — one commit, one reason.
- Good concrete justification: gave a real scenario (needing to revert just the auth fix while keeping the CSS change) rather than just saying "it's cleaner."

**Missing:**
- Didn't flag that "fixes" itself is a bad message on top of being a mixed commit — a good message says what and ideally why, e.g. `fix: reject empty bearer token in auth middleware`, not a one-word placeholder.
- Missed the code-review angle: mixing a security-sensitive auth change with unrelated renames and a CSS tweak makes it much harder for a reviewer to focus attention where it actually matters — bugs hide in the noise.
- Didn't mention `git bisect`: atomic commits mean a future regression search lands cleanly on the one commit that changed the auth logic, instead of a commit that also touched two other unrelated things.

**Tip:** Before committing, ask "if I had to revert just one of these changes next week, could I cleanly?" — if not, split the commit.

---

## Q2 — Core Java & CS Fundamentals

**Question:** A teammate ships a "daily digest" email feature: `LocalDateTime sendAt = LocalDateTime.now().withHour(9).withMinute(0);`. This gets stored and used to trigger sending at "9am." A user on the US West Coast says the email arrives at 6am their time, and a user in Tokyo says it shows up at 6pm the previous day. What's the underlying mistake, and how would you fix the type/design?

**Answer:**
> LocalDateTime has no timezone info, so each user's data ends up with different times which should be the same; proposed switching to OffsetDateTime or Instant

**Score: 6/10**

**Strong:**
- Correctly flagged that `LocalDateTime` carries no timezone/offset information — the right type-level diagnosis.
- Right direction on the fix, naming `OffsetDateTime`/`Instant` as more appropriate types.

**Missing:**
- The mechanism needed more precision: `LocalDateTime.now()` captures the server's wall-clock time with zero zone info attached; once stored, nothing records what zone that "9:00" was measured in.
- Bigger gap: switching to `Instant` alone doesn't fix "wrong local time" for the recipient — `Instant` is a single global moment, not "9am for this specific user." The real fix needs each user's `ZoneId` (e.g. `ZonedDateTime.of(date, LocalTime.of(9,0), user.getZoneId())`), converted to `Instant`/UTC only for storage/triggering.
- Didn't mention storing the recipient's time zone as data (a user preference field) — the type fix and the "whose 9am" design fix are two separate things.

**Tip:** Whenever `LocalDateTime.now()` touches anything cross-timezone, ask "whose 9am is this?" — if the answer depends on who's reading it, you need a `ZoneId` in the picture, not just a UTC-safe type.

---

## Q3 — OOP, Design Patterns & LLD

**Question:** A PR has `abstract class Employee` with subclasses `SalariedEmployee` and `HourlyEmployee`, each overriding `calculatePay()`. The team wants to add `ContractorEmployee` — paid hourly, but with a special tax-withholding calculation unrelated to the other types. Keep extending the hierarchy, switch to composition, or reach for an interface instead of the abstract class? Which OOP pillar does the real work when swapping `calculatePay()` per subtype?

**Answer:**
> switch to composition, create interface `PaymentSalary`, inject implementations like `NormalPayment`/`SpecialPayment` per class

**Score: 6/10**

**Strong:**
- Right call on composition over inheritance — injecting a swappable behavior instead of growing the subclass tree.
- Correctly reaches for an interface + multiple implementations rather than another layer of subclassing.

**Missing:**
- Didn't answer which OOP pillar is doing the work: **polymorphism** — the concrete implementation that runs is decided at runtime through the interface.
- Didn't explain why inheritance breaks down specifically here: `ContractorEmployee` needs to combine "hourly-style pay" with "a unique tax rule," and single inheritance can't cleanly mix two independent behaviors without duplication or an awkward subclass.
- Didn't address interface vs abstract class directly: `Employee` can stay an abstract/base class for shared fields, while the varying piece (pay calculation) is pulled into an interface — the two aren't mutually exclusive.
- Naming nit: `PaymentSalary` names a noun; a strategy interface is usually named for the action it performs (`PayCalculator`/`PaymentStrategy`).

**Tip:** When naming a strategy interface, ask "what verb does this do?" — that verb becomes the interface name and its single method.

---

## Q4 — Databases: SQL, Indexing & Data Modeling

**Question:** A signup service does `SELECT * FROM users WHERE email = ?` in the service layer, and inserts if no row comes back. Under load, two near-simultaneous requests with the same email both slip past the check, producing two rows with the same email. What's missing from the schema that should have made this impossible regardless of the application code, and why doesn't the app-level check alone protect you?

**Answer (after one hint about the timing of the race):**
> proposed Pessimistic Locking, reasoning that concurrent requests can't see each other causing a "dirty read," and that duplicates occur "although we use UNIQUE for email field"

**Score: 4/10**

**Strong:**
- Correct diagnosis of the pattern: two concurrent requests racing past a check-then-insert isn't safe just because the app "checked first" — a real check-then-act race, reached after one hint.
- Understands some extra concurrency mechanism is needed beyond the plain SELECT-then-INSERT.

**Missing:**
- Core misconception: a UNIQUE constraint on `email` **does** prevent this — that's exactly what "unique index as a data guarantee" means. If two inserts race for the same email, the database rejects the second `INSERT` with a constraint violation, regardless of what the app checked beforehand. It's not that UNIQUE "still allows" the duplicate — UNIQUE is precisely the mechanism that makes it structurally impossible.
- Pessimistic locking (`SELECT ... FOR UPDATE`) doesn't fit this case: at the moment of the race, no row with that email exists yet, so there's nothing to lock. Pessimistic locks protect existing rows during updates (lost-update problem), not duplicate inserts of a row that doesn't exist yet.
- Correct fix: UNIQUE constraint on `email`, then catch the resulting constraint-violation exception on insert (e.g. "email already taken") instead of trusting the earlier SELECT.

**Tip:** For any invariant your app must hold, ask "does the schema enforce this, or only my code?" — if only the code, concurrent requests will eventually break it. (A hint was needed to get past the initial framing here — worth revisiting before the real interview.)

---

## Q5 — Spring Boot & JPA/Hibernate

**Question:** A `transferFunds` service method is annotated `@Transactional`. It debits one account, then credits another. Partway through the credit step, a checked `IOException` is thrown (from a logging call). Does the debit get rolled back automatically? Why or why not, and what would you change if you wanted it to?

**Answer:**
> yes it will roll back — "it is the way the @Transactional annotation work ... when 1 statement have error all statement will rollback"

**Score: 3/10**

**Strong:**
- Correct general mental model: `@Transactional` wraps the method in a transaction meant to undo everything on error — right instinct for why this matters in a funds-transfer method.
- Connected the concept to a real consequence (balance consistency).

**Missing:**
- This is the actual point of the question, and the answer given is the opposite of correct: **the debit would NOT roll back here.** Spring's `@Transactional` only rolls back automatically on unchecked exceptions (`RuntimeException`/subclasses) and `Error`. A checked exception like `IOException` is excluded by default — the transaction would commit, the debit would go through with no matching credit.
- To roll back on a checked exception, you must say so explicitly: `@Transactional(rollbackFor = IOException.class)` (or a broader `Exception.class`).
- The checked-vs-unchecked distinction is the single most important gotcha in this topic and was the actual thing being tested.

**Tip:** Memorize as one line: "`@Transactional` rolls back on unchecked exceptions and `Error`s by default — checked exceptions need `rollbackFor` explicitly." Treat any checked exception inside a `@Transactional` method as a flag to check this.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Tooling: Git, Build & Environment — atomic commits in a mixed PR | 7/10 |
| 2 | Core Java & CS Fundamentals — `LocalDateTime` timezone bug | 6/10 |
| 3 | OOP, Design Patterns & LLD — composition over inheritance for pay types | 6/10 |
| 4 | Databases: SQL, Indexing & Data Modeling — unique constraint vs race condition | 4/10 |
| 5 | Spring Boot & JPA/Hibernate — `@Transactional` rollback on checked exceptions | 3/10 |

**Overall: 5.2/10**

**Strengths:**
- Reasons through concrete scenarios rather than reciting definitions — traced the commit-splitting rationale, the composition trade-off, and (once nudged) the signup race condition step by step.
- Comfortable naming the right-sounding constructs (`OffsetDateTime`/`Instant`, interfaces, `UNIQUE`, `@Transactional`) — the vocabulary is there, even where the underlying mechanism wasn't fully right.
- Willing to reason out loud and adjust direction after a small hint rather than freezing up.

**Areas to review before the real interview:**
- `@Transactional` rollback semantics — default only covers unchecked exceptions/`Error`; checked exceptions need `rollbackFor` explicitly. This is the biggest gap this session and a classic interview gotcha.
- Unique constraints as the actual guarantee against duplicate data under concurrency — don't reach for locking before confirming what a schema-level constraint already solves.
- Explaining mechanism, not just outcome — several answers had the right instinct (timezone fix, composition-over-inheritance) but skipped why it works at the level a strong answer needs.
- General pattern: default to trusting schema/database-level guarantees (constraints, transactions) over rebuilding the same protection in application code.
