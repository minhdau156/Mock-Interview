# Mock Interview — Junior Backend Developer
**Date:** 2026-09-02

**Categories covered (3-question session, date-based seed (no timestamp available): seed = 2×137 + 9×31 = 553, start=4, step=9; sequence: Spring Boot & JPA/Hibernate → Databases: SQL, Indexing & Data Modeling → OOP, Design Patterns & Low Level Design; opening category (4) differs from 08/30's opener (9) per the no-repeat-opener rule. Note: the mechanical seed happens to reproduce 08/25's exact category set/order (4→3→2) — since that session isn't the immediately-previous one, the rule isn't violated, but sub-topics within each category were deliberately chosen to avoid any repeat: the raw per-question index landed on `@Transactional` rollback (already tested twice), so it was substituted for the untested `@Value` vs `@ConfigurationProperties` angle; Databases used the untested "status field vs. status-history table" data-modeling angle instead of JOINs/indexing/1NF (all previously covered); OOP used a from-scratch vending-machine LLD exercise, a format never used before in this series.):** Spring Boot & JPA/Hibernate, Databases: SQL, Indexing & Data Modeling, OOP, Design Patterns & Low Level Design

---

## Q1 — Spring Boot & JPA/Hibernate

**Question:** A teammate's `PaymentGatewayClient` class has grown to this:

```java
@Component
public class PaymentGatewayClient {

    @Value("${payment.stripe.api-key}")
    private String apiKey;

    @Value("${payment.stripe.base-url}")
    private String baseUrl;

    @Value("${payment.stripe.timeout-ms}")
    private int timeoutMs;

    @Value("${payment.stripe.retry-count}")
    private int retryCount;

    // ... 4 more @Value fields for related Stripe settings
}
```

Code review comment: "Can you switch this to `@ConfigurationProperties` instead?" The teammate pushes back: "Why bother, `@Value` already works fine?"

How would you answer them — what actually changes if you switch, and is there a meaningful difference in what happens if `payment.stripe.timeout-ms` is missing from the properties file in each approach?

**Candidate's answer** (asked for clarification of what each annotation does; given two hints — first contrasting `@Value`'s per-field injection against `@ConfigurationProperties`' whole-class binding, second pointing specifically at startup-time behavior when a key is missing):
> "if switch that contain all value in file application.yml i think it can work well but when it miss i think when use the value can work well"
>
> "i think the annotation value can throw exception but @ConfigurationProperties can work well"

**Feedback given:**
- Strong: after two hints, landed on the correct core asymmetry — `@Value` fails fast (throws if the placeholder can't be resolved and no default is given), `@ConfigurationProperties` does not throw on a missing key.
- Missing: the *why* — `@Value` throws `IllegalArgumentException: Could not resolve placeholder` unless a fallback like `${...:5000}` is given; `@ConfigurationProperties` uses relaxed/lenient binding, silently leaving a missing field at its Java default (`0` for `int`, `null` for `String`).
- Missing/misconception: framed `@ConfigurationProperties`'s silent behavior as positive ("can work well") without recognizing it's actually a hidden danger — `timeoutMs = 0` with no startup error could cause every Stripe call to fail immediately in production. The real fix is `@Validated` + `@NotNull`/`@Min` on the fields to turn this back into a fail-fast startup error.
- Missing: didn't mention the practical reasons to prefer `@ConfigurationProperties` — grouping related settings into one type-safe class, unit-testability without a Spring context, and support for nested/complex types `@Value` can't bind directly.

**Score: 5/10**

---

## Q2 — Databases: SQL, Indexing & Data Modeling

**Question:** Your team's `orders` table looks like this:

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    status VARCHAR(20),   -- 'PENDING', 'PAID', 'SHIPPED', 'DELIVERED', 'CANCELLED'
    total DECIMAL,
    updated_at TIMESTAMP
);
```

Whenever an order's status changes, the app just does `UPDATE orders SET status = ?, updated_at = ? WHERE id = ?`.

A customer calls support: "I was charged for this order three days ago but it still says PENDING — when did it actually move to PAID, and how long did it sit in PAID before shipping?" Support opens the row and... there's no way to answer that. Just the current status and one `updated_at` timestamp.

What's the actual limitation in this schema, how would you redesign it to answer that kind of question, and what's the cost you're accepting by doing so?

**Candidate's answer** (needed the question broken into three explicit parts, then three hints to reach the root cause — first restating the three parts, second pointing at what `UPDATE` does to the row's previous values, third asking directly yes/no whether the old value survives):
> "so what you want me answer"
>
> "but i see the query is correct can you give me some hint"
>
> "oh in this scope i think we don't need that status PAID"
>
> "No because we override this and it is not a enum"
>
> [redesign] "i think i will add create a new table that track the changing in this order for this specification it should have the id for order, its id and the old status with the updated time so when we change we can know when it is created and the time now and updated time of the status to know how long it keep in the DB"
>
> [cost] "it will cost more memory because we need to create the new table using the index in it and can make our query slow"

**Feedback given:**
- Strong: correctly identified the root limitation once nudged — `UPDATE` overwrites the single `status`/`updated_at` slot, so every prior transition's value is gone the instant the next one is written.
- Strong: landed on the right redesign shape independently (no hint needed) — a separate table keyed by `order_id`, storing each status value with its own timestamp, so a transition becomes an `INSERT` instead of an `UPDATE`.
- Missing: the enum-vs-VARCHAR tangent was a non-sequitur — an `ENUM` column gets overwritten by `UPDATE` exactly the same way a `VARCHAR` one does; that distinction is about which strings are allowed, not about whether old values are preserved.
- Missing/misconception: the trade-off answer stayed vague ("more memory," "query slow") instead of naming the actual mechanism — writes get more expensive (every transition now needs an `INSERT` into history, plus possibly a second write to keep a denormalized `orders.status` in sync for fast current-state reads), and reading "current status" gets harder without that denormalized column (requires `ORDER BY changed_at DESC LIMIT 1` against the history table). Indexing the history table is cheap and standard — it wasn't the real cost here.
- Needed the question broken into three explicit parts and three hints before reaching the core insight, compared to answering the redesign and cost sub-questions directly once reached.

**Score: 4/10**

---

## Q3 — OOP, Design Patterns & Low Level Design

**Question:** Let's do a design exercise instead of a bug-spotting one this time.

Your manager says: "We're building the backend for a simple **vending machine** — nothing fancy. A customer selects an item by slot code (like `A3`), inserts money, and if they've inserted enough, the machine dispenses the item and returns change. If the slot is empty or they haven't inserted enough, it should tell them that instead."

Walk me through your design:
- What are the main classes/interfaces you'd create, and what does each one own?
- Where does the state live — what data changes as the machine gets used, and what's fixed/static?
- Is there a piece of this that varies enough to justify an interface (e.g. different payment methods), or would that be over-engineering for what's described here?

**Candidate's answer** (asked one scoping question first — whether the machine serves one customer at a time, confirmed yes — then answered fully unprompted, no hints needed):
> "so first i think i need to know some of class like this
>
> class Drink have some variable like id, type, quantity, price, code
> concrete class like Pepsi, Coca, Orange juice, Coffee, Tea ... inherit this class
>
> so i will need a Manager which use a singleton pattern
> and it will handle the price user insert and forward to the another class to handle the drink and the price
>
> the DrinkHandler class have the method that accepted the code user type in in Manager and the cost they insert to this machine
>
> i think i should have some attribute in Manager class like state it should have Idle, Working
>
> if in working state the machine are in the transaction with another user and refuse the user,
>
> so in the manager we should have the method that will handle the state of the machine it should like this
>
> and in the DrinkHandler it should have the collection of Drinks so
> the workflow should like this we should have the method that choose the payment method first in the Manager class
>
> and we should the interface like PaymentStrategy and another concreteclass should implemnt it like CashPayment, CardPayment, .....
>
> and we inject the PaymentStrategy to the DrinkHandler so we can create new Object of each payment method and use it in the method we handle the drink
>
> the all workflow like the
> choosePaymentMethod (in Manager Class) -> getDrink(String code, String paymentMethod) (in Manager Class) -> getDrink(String code, String paymentMethod) (in DrinkHandler) in this method we do
> - create the Object of PaymentStrategy
> - get the Drink base on the code we provide in the argument
> - and we will get the cost in the Object of PaymentStrategy it should the method and the variable that get the money and the variable hold the money
> - and we will check with the price withe Drink if it enough we will get it and drop it to the User
> - otherwise log a message to user"

**Feedback given:**
- Strong: fully self-directed, no hints needed, covered most of what was asked without prompting.
- Strong: correctly identified payment method as the piece that genuinely varies in behavior (not just data) and modeled it with a `PaymentStrategy` interface (`CashPayment`, `CardPayment`) — the right call, contrasted against the `Drink` hierarchy misstep below.
- Strong: `Idle`/`Working` state on the machine-level class, refusing new transactions mid-purchase, correctly models the one-customer-at-a-time constraint from the earlier scoping question.
- Missing/misconception: modeling `Pepsi`/`Coca`/`OrangeJuice`/`Tea` as subclasses of `Drink` is over-engineering — these differ only in data (name, price, code), not behavior; a single concrete `Drink` class with different field values per instance is correct, the same principle the topic pool calls out directly for "different drink types that only differ in name/price."
- Missing/misconception: conflated "inject" with "create a new object based on a string code" — mapping a `paymentMethod` string to the right `PaymentStrategy` instance at runtime is a Factory's job; injection means the caller (e.g. Spring) already hands over a built instance. What was described is Strategy + Factory together, not "inject."
- Missing: money/change state was never pinned down — where "amount inserted so far" lives during a transaction, and how change is calculated and returned, were both left vague ("the variable that get the money and the variable hold the money").
- Missing: stock/out-of-stock handling was never addressed, despite being explicitly asked in the prompt — `Drink.quantity` exists but nothing decrements it or checks it before dispensing.
- Missing: `Manager` and `DrinkHandler` have unclear, overlapping responsibility — `Manager.getDrink(...)` just forwards to `DrinkHandler.getDrink(...)` with the same signature.

**Score: 6/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | Spring Boot & JPA/Hibernate — `@Value` vs `@ConfigurationProperties`: fail-fast vs. silent lenient binding on a missing property | 5/10 |
| 2 | Databases: SQL, Indexing & Data Modeling — a single `status` column can't answer "when did this transition happen"; redesigning to a status-history table | 4/10 |
| 3 | OOP, Design Patterns & Low Level Design — designing a vending machine: classes, state, and where an interface is actually justified | 6/10 |

**Overall: 5.0/10**

**Strengths:**
- Delivered a comprehensive, largely self-directed answer to the LLD exercise (Q3) with zero hints, correctly identifying payment method — not drink flavor — as the piece of the system that actually varies in behavior and deserves a `Strategy` interface.
- Once nudged, reasoned through *why* a mechanism behaves the way it does rather than just restating the fact — e.g., correctly connecting "UPDATE overwrites the row" to "so the old value is gone," not just asserting it.
- Reached the correct schema redesign (an insert-only history table) independently, without needing a hint for that specific part.

**Areas to review before the real interview:**
- **Commit to a specific guess before asking for a hint.** Several answers this session opened with "what do you want me to answer" or "can you give me a hint" before attempting anything concrete — a wrong, specific guess is both faster to correct and shows more reasoning than asking for direction first.
- **Inheritance is for behavioral variation, not data variation.** Modeling `Pepsi`/`Coca`/`Tea` as `Drink` subclasses when they only differ in name/price is the same shape of over-engineering the topic pool explicitly calls out — watch for this reflex whenever "different types of X" shows up in a design prompt.
- **Keep Factory and Dependency Injection conceptually separate** — both can appear in the same design (a factory maps a string to a `Strategy` implementation; DI is how a caller receives an already-built dependency), but they solve different problems and answers should name which one is actually happening.
- **Vague trade-off language** ("more memory," "query slow," "can work well") should be pushed into a concrete mechanism — which specific write, which specific query, which specific field. This showed up in both Q1 and Q2.
- **Read every explicit requirement in the prompt before designing** — Q3 directly asked about returning change and handling an empty slot, and neither made it into the design. A quick checklist pass against the prompt's stated requirements before finalizing an answer would catch this.
