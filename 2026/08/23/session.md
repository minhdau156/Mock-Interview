# Mock Interview — Junior Backend Developer
**Date:** 2026-08-23

**Categories covered (3-question session, date-based seed (no timestamp available): seed = 23×137 + 8×31 = 3399, start=10, step=9; sequence from start: Tooling: Git, Build & Environment → System Design & Architecture Basics → Web Security & Frontend Basics; opening category (10) differs from 08/21's opener (6) per the no-repeat-opener rule; all three scenarios were checked against every prior session back to 07/04 to avoid reusing a scenario/code snippet already asked — Git Tooling avoided commit-hygiene/secrets/dependency-conflict/merge-conflict/rebase/PR-pushback angles already used, System Design avoided CRUD-layering/caching/URL-shortener/sync-vs-async/stateless-sessions/scaling/controller-service-repo angles already used, Web Security avoided request-lifecycle/CORS/cookie-JWT/session-vs-token/SQL-injection/useEffect-loop/hidden-button angles already used):** Tooling: Git, Build & Environment, System Design & Architecture Basics, Web Security & Frontend Basics

---

## Q1 — Tooling: Git, Build & Environment

**Question:** Your project's `pom.xml` has this:

```xml
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.24.2</version>
    <scope>test</scope>
</dependency>
```

A teammate opens a PR that uses `Assertions.assertThat(...)` from AssertJ directly inside `OrderService.java` — real production code, not a test — to validate an order's total before saving it. It compiles fine on their machine, all tests pass in CI, and the PR gets approved. Two days after deploying to prod, the service throws `NoClassDefFoundError: org/assertj/core/api/Assertions` the first time that code path runs.

Walk me through: why did this pass locally and in CI but blow up in prod, and what should the reviewer have caught?

**Candidate's answer** (asked for a hint calling the question "too vague"; given a nudge pointing at what `<scope>test</scope>` actually promises about classpath availability):
> "ok i understand about that the tag scope it just resolve the dependency in test environment so when add this in this OrderService it can just compile fine but in prod environment it can not resolve this dependency so that the error occur / so i think if we want to use it we should create OrderServiceTest and use this in it it seperate each environment so we can test it and can run it properly in local and prod"

**Feedback given:**
- Strong: correctly identified the core mechanism — `test` scope only puts the dependency on the test classpath, not the packaged runtime classpath the deployed JAR actually runs with.
- Strong: correctly proposed the fix direction — this code shouldn't be exercised outside a test at all.
- Missing: what the reviewer should have flagged specifically — an import from `org.assertj.*` inside `src/main/java` is itself the red flag, independent of whether CI is green.
- Missing: why it looked fine locally/in CI — IDEs commonly merge test + main classpaths for a single module, and CI running `mvn test` has the test classpath active for the whole build; only packaging/running the artifact the way prod does (compile+runtime scope only) surfaces the missing class.
- Missing: a sharper fix than "move it into a test class" — AssertJ's `assertThat(...)` throws `AssertionError`, meant to fail a test, not to be business-rule control flow in a running service; production validation needs a real domain exception instead.
- Needed a hint to get past "too vague" — noted for the "commit to a guess before asking for the answer" pattern.

**Score: 6/10**

---

## Q2 — System Design & Architecture Basics

**Question:** Your team's checkout flow currently has one concrete need: after an order is placed, send a confirmation email. A teammate opens a PR with this:

```java
public abstract class AbstractOrderPostProcessor<T> {
    protected abstract T extractPayload(Order order);
    protected abstract void beforeProcess(T payload);
    protected abstract void process(T payload);
    protected abstract void afterProcess(T payload);

    public final void run(Order order) {
        T payload = extractPayload(order);
        beforeProcess(payload);
        process(payload);
        afterProcess(payload);
    }
}

public class EmailConfirmationProcessor extends AbstractOrderPostProcessor<EmailPayload> {
    protected EmailPayload extractPayload(Order order) { ... }
    protected void beforeProcess(EmailPayload p) { }   // empty, nothing to do
    protected void process(EmailPayload p) { emailService.send(p); }
    protected void afterProcess(EmailPayload p) { }    // empty, nothing to do
}
```

Their reasoning: "This way, when we add SMS or push notifications later, we just extend the base class." Right now, email is still the *only* post-processing step that exists. What would you say about this design in review?

**Candidate's answer** (asked for a hint; given a nudge to reuse their own past reflex — "what's actually varying right now, and how often?"):
> "i think it is the Strategy Pattern and with 4 method but right now it just have 1 implementation so the pattern it have the code more complex and redundant so and it have 2 method are empty so in my case i will remove the abstract class and 2 method in it"

**Feedback given:**
- Strong: correct bottom-line judgment — with one implementation and two no-op hooks, the abstraction is premature and should be removed rather than kept "for later."
- Strong: correctly flagged the empty `beforeProcess`/`afterProcess` methods as a concrete symptom, not just a vague "it's redundant."
- Missing: misnamed the pattern — this is **Template Method** (fixed algorithm skeleton, subclasses fill in steps), not **Strategy** (swap a whole algorithm via an injected/composed object, no shared base class dictating order). A recurring imprecision.
- Missing: what to replace it with concretely — a plain `EmailConfirmationService.sendConfirmation(order)` method, no abstract class or hooks, until a second real implementation exists.
- Missing: the actual trigger for reintroducing the abstraction later — when a second post-processor lands with genuinely different before/after behavior, informed by what two real implementations have in common.

**Score: 5/10**

---

## Q3 — Web Security & Frontend Basics

**Question:** Your team's React "Create Product" form has this:

```jsx
function CreateProductForm() {
  const [price, setPrice] = useState('');

  function handleSubmit(e) {
    e.preventDefault();
    if (price === '' || Number(price) <= 0) {
      setError('Price must be a positive number');
      return;
    }
    api.post('/products', { price: Number(price), name: ... });
  }
  // ... renders an input bound to `price`, shows `error` under it
}
```

A teammate reviewing the PR says: "Good, that covers the validation — empty or zero/negative prices are blocked before we even hit the network." The backend `POST /products` endpoint just does `productRepository.save(new Product(dto.price(), dto.name()))` with no checks of its own. Do you agree the validation is "covered"? What's the actual risk, and what needs to change?

**Candidate's answer** (asked what `Number(...)` does; told it converts the input string to a numeric value, e.g. `"abc"` → `NaN`):
> "it don't check the case that the price is Nan is like undefine so when go to the backend and the backend not validate again it will save to DB and it is the data that we don't want to save so we need to add use some function to check does it is a number or not first and then 2 condition like in the question"

**Feedback given:**
- Strong: genuinely sharp, non-obvious catch, found without a conceptual hint — `Number("abc")` is `NaN`, and `NaN <= 0` evaluates to `false` in JavaScript, so the frontend's own guard silently lets a garbage string through as "valid."
- Strong: correctly traced the consequence through to `productRepository.save(...)` writing the bad value to the database.
- Missing: the bigger structural point — even a fully-fixed frontend check wouldn't matter for security/data-integrity, since nothing requires a client to go through this React form; anyone can `POST /products` directly via curl/Postman/devtools, bypassing the JavaScript entirely.
- Missing: the named general rule — a frontend check is a UX convenience, never a validation boundary; the backend must independently re-validate every field on every request.
- Missing: the concrete backend fix — Bean Validation annotations on the request DTO (`@Positive`, `@NotBlank`) plus `@Valid` on the controller parameter, so a bad payload gets rejected with `400` before `save()` — not just patching the same bypassable frontend function.

**Score: 6/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Tooling: Git, Build & Environment — `test`-scoped dependency used in production code, `NoClassDefFoundError` in prod | 6/10 |
| 2 | System Design & Architecture Basics — premature Template Method abstraction for a single post-processor | 5/10 |
| 3 | Web Security & Frontend Basics — frontend-only price validation, `NaN` slip-through, backend trusts client input | 6/10 |

**Overall: 5.7/10**

**Strengths:**
- Correctly diagnoses root mechanisms once given a starting thread — scope semantics, and a specific validation-logic bug (`NaN <= 0`) — without needing the full answer handed over.
- The YAGNI/premature-abstraction instinct ("what's actually varying right now?") continues to be reliable even when the pattern name attached to it is wrong.

**Areas to review before the real interview:**
- **Pattern precision, again**: labeled a Template Method (fixed algorithm skeleton, subclasses fill in steps) as Strategy (swap a whole algorithm via composition). Before naming a pattern, check "is behavior being swapped via an injected object (Strategy), or is a fixed sequence being filled in by subclassing (Template Method)?"
- **"Never trust client input" as a general rule, not just a case-by-case catch**: correctly found the specific `NaN` bug this session and correctly found the hidden-button bypass in a past session, but hasn't yet stated the general principle unprompted — any client-side check, however correct, is bypassable by calling the API directly.
- **Test-scope vs main-source boundary**: got the scope mechanics right, but didn't name what a reviewer should catch on sight (a test-only import inside `src/main/java`) before even running the build.
- Two of three questions needed a hint before an independent answer emerged — try voicing a guess tied to a specific keyword in the question first (as done well for `scope` here) rather than asking "give me a hint" cold.
