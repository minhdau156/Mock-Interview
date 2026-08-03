# Mock Interview — Junior Backend Developer
**Date:** 2026-08-03

**Note:** Per user request, categories were hand-picked (not seed-randomized) to target the least-tested ground across all prior sessions — REST API (3x), OOP/LLD (4x), and Core Java (4x) had the lowest coverage counts in `KNOWLEDGE_SUMMARY.md`, with Web/Frontend and System Design as the next tier by recency. Every question's concrete scenario was constructed to be genuinely different from anything already written up. Per user request mid-session, this session was shortened to 3 questions instead of the usual 5 (REST API, OOP/LLD, and Core Java were covered; Web/Frontend and System Design were not reached).

**Categories:** REST & API Design → OOP, Design Patterns & LLD → Core Java & CS Fundamentals.

---

## Q1 — REST & API Design

**Question:** You're reviewing a teammate's PR for `POST /api/orders/{id}/cancel`. Right now the endpoint always returns `200 OK` — whether the cancel actually succeeded, or the order ID doesn't exist, or the order was already shipped and can't be cancelled. Every case just comes back as `200` with a JSON body like `{ "success": false, "error": "order not found" }`, and the frontend has to parse that field every single time to know what actually happened. What's wrong with this design, and what would you change about the status codes (and the response shape) for these three outcomes?

**Candidate's answer:** the design is wrong on the response status code in each condition it always returns 200 OK so when have the error it hard to see the cause and it hard for the frontend like you say in the question; so for me i will return the status code different between 3 condition; the cancel actually succeeded it should have the status code 200 `{"success": true, "error": null, "response": "Your order are cancelled successfully"}`; the order ID doesn't exist it should have the status code 404 `{"success": false, "error": "Order ID not found", "response": null}`; the order was already shipped and can not be cancelled it should have the status code 409 `{"success": false, "error": "The order are shipping, can not canceled", "response": null}`

**Score: 7/10**

**Strong:**
- Correctly diagnosed the actual problem: always-200 makes success and failure indistinguishable without parsing the body every time.
- Nailed `404` for "order not found."
- Nailed `409` for "already shipped, can't cancel" — this is the one juniors most often get wrong (many reach for `400 Bad Request` here). Choosing `Conflict` specifically shows real precision: the request itself is well-formed, it's just that the *current state* of the resource conflicts with the action being requested.

**Missing:**
- The three response bodies are shaped inconsistently case-by-case (`response` holds a success message in one, is `null` in the others). In a real API you want one single envelope shape used by every endpoint, not one invented per scenario. In Spring, this is normally solved with a global `@ControllerAdvice` + `@ExceptionHandler` that catches specific exceptions (`OrderNotFoundException`, `OrderAlreadyShippedException`, etc.) and maps each to the right status + a consistent JSON error shape.
- A production error body usually also carries a stable, machine-readable error code (e.g. `"code": "ORDER_ALREADY_SHIPPED"`) separate from the human-readable message — the frontend can safely switch on that code even if the message text changes later, but can't safely pattern-match on the string itself.

**Tip:** Next time you design an error response, sketch the envelope shape once as a shared DTO first, then ask "does every failure case fit into this same shape?" instead of writing each case's JSON independently.

---

## Q2 — OOP, Design Patterns & LLD

**Question:** Say you're asked to design the core classes for a vending machine — coins go in, a button gets pressed, a product either dispenses or an error occurs (insufficient funds, out of stock). Walk me through the main classes/interfaces you'd create and how they relate to each other. What state does the machine need to track, and where does that state live?

**Candidate's answer:** i think first i will create the VendingManager class which will have 2 DI: FormulaManager for managing the coin get in the machine, DrinkFactory for dispense the drink that customer choose; and the Drink interface which have 3 field code, price, name and the class Drink that implement the Drink class like Pepsi, Coca, Tea, Coffee; when you get the coin in and type the code of drink the FormulaManager will handle whether you have enough fund or insufficient fund it will throw the error and the Vending Manager will handle the error; so if you have enough we will go to the DrinkFactory to dispense the drink and it will match the suitable code if it have give it to customer else out of stock or no code matching throw exception so VendingManager will handle it

**Score: 4/10**

**Strong:**
- Reasonable separation-of-concerns instinct: a top-level orchestrator (`VendingManager`) delegating to a component that handles money and a component that handles dispensing, instead of one giant class doing everything.
- Correctly identified the two failure modes worth handling: insufficient funds and out-of-stock.

**Missing:**
- Didn't actually answer the state half of the question. A vending machine needs at least two very different kinds of state: (1) the current inserted balance for the transaction in progress (resets after each purchase or refund), and (2) per-drink stock counts that persist across transactions and get decremented on dispense. Naming these explicitly, and saying which object owns each, is the core of this exercise.
- "`Drink` interface implemented by Pepsi, Coca, Tea, Coffee" reaches for an interface + multiple implementers where nothing actually varies in behavior between drinks — they only differ in data (code/price/name). This is exactly the YAGNI-style question already applied correctly in past sessions ("what specifically is going to vary, and how?"); here the answer is "nothing behaviorally," so `Drink` should just be one concrete class (or record), and Pepsi/Coca/Tea/Coffee are instances of it, not separate implementing classes.
- Calling the dispensing component a "Factory" mixes two different jobs. A Factory's role (in the pattern sense) is deciding which object to create. Matching a code to a slot, checking quantity, and decrementing stock on dispense is inventory/repository work — something like `Inventory` or `StockManager` would be more accurate. Same precision gap flagged before: Factory ≠ "a class with a method that does stuff."
- Not required at junior depth, but worth knowing it exists: this exact exercise is a classic teaching example for the State pattern (`IDLE` → `HAS_FUNDS` → `DISPENSING`, or a `SOLD_OUT` state that rejects coins) — noticing "the machine's overall mode is itself a piece of state" would have been a strong thing to mention.

**Tip:** For any LLD exercise, list your nouns (classes) and your state variables separately before writing behavior — "what does this object know" is a different question from "what does this object do."

---

## Q3 — Core Java & CS Fundamentals

**Question:** A teammate writes a cache using a custom `RequestKey` class as the `HashMap` key — it has two mutable fields (`userId`, `timestamp`) that get updated after the object is already inserted into the map. Lookups start silently failing (`map.get(key)` returns `null` even though the same object is still sitting in the map). Meanwhile, elsewhere in the codebase, `String` is used as a `HashMap` key constantly and this problem never comes up. What's actually going wrong with `RequestKey`, and why does `String` not have this problem?

**Candidate's answer:** (after one hint, then additional scaffolding, then a second nudge) i think when get it will jump to specific bucket on hashCode() and execute but the key is different than so it not get but the String is immutable so when use hashCode it will give the same key so it no problem

**Score: 5/10**

**Strong:**
- With scaffolding, landed on the actual mechanism: `get()` recomputes `hashCode()` on the key you hand it, right now, and jumps to whatever bucket that current hash points to — since `RequestKey`'s fields changed after insertion, that's a different bucket than the one the object actually lives in, so the lookup misses it even though the object is still in the map.
- Correctly connected `String`'s immutability to why this never happens with `String` keys.

**Missing:**
- Took two rounds of hints to get here — worth internalizing as a named mechanism stated cold, since a real interview won't offer scaffolding.
- Didn't fully land the other half of the "why": `HashMap` never rehashes or relocates an existing entry when its key later mutates — the entry stays physically parked in the bucket its original hash pointed to, forever. So the object isn't lost or corrupted; it's just unreachable via `get()` because only the lookup side recomputes the hash, and now it's asking a different bucket for it.
- Bonus detail worth knowing: `String` doesn't just "happen" to keep the same hash — Java caches the computed `hashCode()` inside the `String` object the first time it's called, which immutability makes safe to do.
- Reusable lesson to state explicitly: never mutate a field that `equals()`/`hashCode()` depends on while the object is live as a `HashMap`/`HashSet` key — mutable-field-in-hash-key is the actual code smell, not "HashMap is buggy."

**Tip:** Build this reflex: whenever you design a class that will be used as a Map/Set key, ask "are all the fields involved in `equals()`/`hashCode()` effectively immutable for as long as this object might live as a key?" If not, either make them immutable or don't use that type as a key.

---

## Session Summary

| # | Topic | Score |
|---|-------|-------|
| 1 | REST & API Design — per-outcome status codes & consistent error envelope | 7/10 |
| 2 | OOP/LLD — vending machine class design & state modeling | 4/10 |
| 3 | Core Java — mutable HashMap keys vs. String immutability | 5/10 |

**Overall: 5.3/10**

**Strengths:**
- Sharp status-code judgment — correctly reached for `409 Conflict` for a state-conflict error rather than the common junior mistake of `400`.
- Reasonable separation-of-concerns instinct in class design (delegating money-handling and dispensing to separate components).
- Once given enough scaffolding, reasons through to the correct low-level mechanism (HashMap bucket lookup) rather than guessing.

**Areas to review before the real interview:**
- **Consistent, centralized error-response contracts.** Design one shared error/response shape and produce it via a global mechanism (Spring's `@ControllerAdvice`/`@ExceptionHandler`), not one invented per endpoint — plus include a stable machine-readable error code alongside the human message.
- **LLD process discipline.** Explicitly separate "what state does this system track" from "what classes exist" before designing, and apply the already-demonstrated YAGNI instinct consistently — don't reach for an interface + multiple implementers when types only differ in data, not behavior.
- **Design-pattern vocabulary precision.** A recurring gap: naming something a "Factory" when its actual job is inventory/repository management (tracking and decrementing stock) rather than deciding which object to create.
- **HashMap key-mutation mechanics, stated unprompted.** Be able to say cold, without hints: an entry's bucket is fixed at insertion time; only lookups recompute the hash, so mutating a key's hash-relevant fields makes it silently unreachable, not corrupted or lost.
