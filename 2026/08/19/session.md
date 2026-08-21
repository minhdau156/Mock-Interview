# Mock Interview — Junior Backend Developer
**Date:** 2026-08-19

**Categories covered (3-question session, date-based seed (no timestamp available): seed=2851, start=2, step=3; sequence from start: OOP/Design Patterns & LLD → REST & API Design → Web Security & Frontend Basics → Core Java & CS Fundamentals → Spring Boot & JPA/Hibernate → Testing & Debugging → ...; the 08/17 session covered Web Security, Core Java, and Spring Boot/JPA, so those three were skipped to avoid repeats, landing on OOP/Design Patterns & LLD, REST & API Design, Testing & Debugging; opening category (2) differs from 08/17's opener (8) per the no-repeat-opener rule):** OOP, Design Patterns & Low Level Design, REST & API Design, Testing & Debugging

---

## Q1 — OOP, Design Patterns & Low Level Design

**Question:** You're reviewing a PR. To send order-confirmation emails, the author wrote this:

```java
interface NotificationStrategy {
    void send(Order order);
}

class EmailNotificationStrategy implements NotificationStrategy {
    public void send(Order order) { /* send email */ }
}

class SmsNotificationStrategy implements NotificationStrategy {
    public void send(Order order) { /* send SMS */ }
}

class NotificationStrategyFactory {
    NotificationStrategy create(String type) {
        if (type.equals("EMAIL")) return new EmailNotificationStrategy();
        if (type.equals("SMS")) return new SmsNotificationStrategy();
        throw new IllegalArgumentException("Unknown type: " + type);
    }
}
```

Today, the app only ever sends order-confirmation emails — SMS isn't wired up anywhere, there's no config for it, no ticket asking for it. What would you say in the review, and how do you decide when introducing a pattern like this is worth it versus overkill?

**Candidate's answer (after one nudge to redirect back to the review-comment question):**
> "i think it too much for this because we don't use the SMS at all so i will say in PR that we need to remove the SMSNotification class because it unused in the code and it very redundant so when apply any pattern i will limit it like if it has more than 3 class i will use the strategy pattern or something else if not less than this i will keep it like that for example if it just have 1 class implementation we just use it don't need any strategy pattern"

**Feedback given:**
- Strong: correct core call — introducing `NotificationStrategy` + `NotificationStrategyFactory` for a feature that only ever sends email is overkill; would flag it in review.
- Strong: good instinct to specifically call out the unused `SmsNotificationStrategy` class as dead code to remove.
- Missing: needed a nudge to return to the actual review-comment question.
- Missing: the "more than 3 classes → use the pattern" rule is arbitrary. The real signal is whether variation is real and active (2+ implementations actually in use) or there's a concrete, committed reason a second one is coming — not "someday we might."
- Missing: what's lost by keeping it simple — with one implementation, a plain `EmailSender` class called directly would do, no interface/factory needed. This is premature abstraction: paying a complexity cost today for flexibility not yet needed.
- Missing: the review comment could suggest collapsing to one concrete class now, reintroducing `Strategy`/`Factory` when SMS is actually being built.

**Score: 6/10**

---

## Q2 — REST & API Design

**Question:** You're building `GET /orders` for an admin dashboard. In testing with a handful of seed orders, it works fine — returns a JSON array of every order. Three months after launch, a big customer's order history has grown to 40,000 rows, and now that same endpoint call is timing out and the admin dashboard tab hangs trying to render it.

A teammate suggests: "Let's just add `LIMIT 100` in the SQL query and call it done." What's wrong with that fix on its own, and what would you actually build instead?

**Candidate's answer:**
> "the problem is we can know all the order and the index that it need in , so when we just fetch 100 order and if it fetch 100 old order we don't know the order that already create in this day so it is a problem / so we need to pagination it using the offset or cursor so we can controll all the order and don't miss any order / but i prefer the cursor approach because it can query faster"

**Feedback given:**
- Strong: correctly identified that `LIMIT 100` alone makes most of the data permanently inaccessible — no way to ever request rows 101+.
- Strong: correctly proposed pagination and named both real approaches (offset-based and cursor-based), with a stated preference (cursor) and a reason attached.
- Missing: the precise mechanism for why cursor/keyset pagination is faster — `OFFSET 39900 LIMIT 100` forces a scan-and-discard of 39,900 rows first; a cursor query (`WHERE id > :lastSeenId ORDER BY id LIMIT 100`) uses the index to seek directly, flat performance regardless of depth.
- Missing: `LIMIT 100` with no `ORDER BY` gives no guarantee about which 100 rows come back — pagination needs a stable sort column (`id` or `created_at`) underneath it.
- Missing: the concrete API contract change, e.g. `GET /orders?cursor=<lastId>&size=50` returning the page plus a `nextCursor`.

**Score: 7/10**

---

## Q3 — Testing & Debugging

**Question:** A teammate pings you: "Production is throwing this and I don't know where to even start."

```
java.lang.NullPointerException: Cannot invoke "String.trim()" because the return value of "com.acme.orders.OrderService.getCustomerEmail(Order)" is null
	at com.acme.orders.NotificationSender.send(NotificationSender.java:42)
	at com.acme.orders.OrderController.checkout(OrderController.java:87)
	at org.springframework.web.method.support.InvocableHandlerMethod.invoke(InvocableHandlerMethod.java:150)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:...)
	... 40 more lines of Spring internals ...
```

How would you read this stack trace to figure out where to actually start looking, and what would you check first?

**Candidate's answer:**
> "in my case in the local developmetn i usually have this but in the first i will read the Exception it log in the terminal and read the message so in the message it express very detail that the Stringtrim() can not invoke because the method getCustomerEmail in order service return null / so in that we should check the method that call the trim() first and check the getCustomerOrder later, so we can know which method is wrong and it render in this log in NotificationSender we should check it and handle the case if the email is null we can not to use the trim() method"

**Feedback given:**
- Strong: correctly used the detailed NPE message itself to identify exactly what was null and what operation failed.
- Strong: correctly landed on `NotificationSender.java:42` as the starting frame and proposed a null-guard fix.
- Missing: didn't state explicitly the skill being tested — consciously skipping the `org.springframework.*` frames to find the top-most frame that's your own code, even though the right line was found.
- Missing: stopped at a symptom fix ("don't call `.trim()` on null") without asking why `getCustomerEmail(Order)` returned null in the first place — a guest checkout with no email? a bad migration? a bug upstream?
- Missing: no mention of correlating with the actual failing order/request to reproduce it, since production isn't as reproducible as local dev.

**Score: 6/10**

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | OOP, Design Patterns & LLD — Strategy/Factory over-engineering (YAGNI) | 6/10 |
| 2 | REST & API Design — pagination (offset vs. cursor) | 7/10 |
| 3 | Testing & Debugging — reading a stack trace / NPE triage | 6/10 |

**Overall: 6.3/10**

**Strengths:**
- Consistently reaches the correct core diagnosis and fix direction across very different topics (design, API, debugging) without needing more than one nudge total.
- Good default instincts: flag dead/unused code, prefer cursor pagination, read the exception message carefully before guessing.

**Areas to review before the real interview:**
- Naming the precise mechanism, not just the right answer — *why* cursor pagination scales (index seek vs. scan-and-discard), *why* skip framework frames in a stack trace (not your code), *why* an abstraction is premature (no real second variant yet).
- Root cause vs. symptom fix — the debugging answer stopped at "guard against null" without asking why the value was null in the first place; make that second question a habit.
- Watch for staying on the actual question asked under a nudge — Q1 needed a redirect back to the original review-comment ask.
